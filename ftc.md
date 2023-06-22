# Building FTC

## Introduction
I have been a League of Legends player for years, and a few years ago I came across a fascinating program
made by the community called [cslol-manager](https://github.com/LeagueToolkit/cslol-manager). It allows for custom character 
models to be loaded into the game. When I dove into the code it went way over my head. A couple of months ago I decided it would
be fun to try to make my version that addresses some of the issues I had. After all, what is the point
of having custom character models if your friends can not see them as well? I decided to start from scratch making
it in Rust.

## The Plan
Unfortunately, it is not as simple as switching out the model files. There is a series of checksums and signatures
on each of their models. Change out a file the game will not load. So we need to swap the file's that the game loads at run
time. This should be the easy part as we can just change what file is opened when the game makes the system call to open
the file. The verification checks are the hard part as we will have to overwrite the instructions that verify it is the
proper file and maintain the proper checksums. Luckily the other project had figured this part out, so some of their ideas
can be used to save a lot of time.

## LOL's Defense Mechanism
League of Legends is a competitive game, therefore it has a high priority to prevent cheaters. As such, I soon realized
how hard this was going to be. The first thing I tried was running the code through a disassembler only to find
out that the code was encrypted and gets decrypted at runtime. Welp, time to load up WinDbg then... oh, they also have
debugger checks that will crash the game if there is a debugger attached? I guess this is going to be a pain.
After watching a couple of video game hacking videos
on YouTube, I came across the idea of hooking into Win32 functions that can have their address known by querying Windows
with GetProcAddress. These videos also use LoadLibrayA to load a DLL with all of its functionality. Everything that
seems simple I soon realize the game already has a counter to it. They block loading remote libraries as well
as the CreateRemoteThread, something slightly more complicated. Now it's time to get creative.

## A Solution to Running Code
After days of brainstorming, I came up with my favorite solution. League of Legends prevents using certain functions
that modify their process. However, I am allowed to use WriteProcessMemory, the backbone of the idea. This is
not all I need to get my code to run in their process. I went through the Windows documentation and a couple
of functions caught my eye. GetThreadContext and SetThreadContext, functions that are traditionally used for debuggers
allow me to set RIP to whatever code position I want. By writing assembly code that will execute the instructions I want
and then jump back to the value of RIP before I changed it while keeping all registers intact, I can effectively hijack one
of their threads to do the work for me.

## Writing Assembly
The goal is to write a "procedure" that does all of my work and then jumps back to an arbitrary position while maintaining
all of the registers. When starting this project, League of Legends still ran in 32-bit support mode. Therefore my instruction 
set was limited because jumps to a value specified by a memory offset had not yet been introduced. At first I didn't think it 
was possible to jump to a variable location without using any of the registers. Then
I realized this was a perfect time to use self-modifying code! By using a 32-bit jump to a relative location, I can do the
math of the offset that is required to jump, and then write the offset into the instruction itself. When the CPU reaches that
instruction, it will execute the jump without looking at any of the registers, so I can set the registers to the state they
were in before changing RIP.

## Elegant Shell Code
Traditional shell code is written in assembly. Writing all of my shell code in assembly would be foolish because I needed
to implement complex algorithms. I had to make a decision on what language the shell code would be written in. Should I keep it all in Rust, or use another 
language and potentially increase the complexity of the project? The flaw with writing it in Rust is that my problem is inherently unsafe.
Interfacing with assembly, calling conventions, pointer arithmetic, casting integers to function pointers, and calling Windows API functions
directly. Consequently, I chose to go with C, a choice I am super glad I made. The problem now was how do I get my compiler to
output something useful to me? How about making it produce a Linux ELF dynamic library for the matching architecture?
I mentioned earlier that I cannot compile to a DLL as LoadLibraryA is blocked by the game. However because I can utilize WriteProcessMemory,
I can allocate segments, parse the Linux ELF file, and map the ELF segments from the file into the process memory.
This requires that the output is position independent with the option -fPIE and -fPIC, because I do not know where the process will
be mapped at runtime. Also because I am compiling a Linux executable to use on Windows the standard library does not make sense to
use. Therefore -nostdlib was also provided to ensure no name conflicts were being linked. The [goblin](https://github.com/m4b/goblin)
crate helped as it provided a high-level API for working with ELF files, which allowed me to focus on mapping segments
rather than the specifics of the file format. Once the code was compiled and mapped
into the address space of League of Legends, all I needed to do was specify the calling convention of the main function,
apply the no callee saved registers attribute, and call the function from the assembly written earlier.

## Side Effects in Shell Code
It is impossible to know at compile time the address of the Windows API functions that we need to call. For example to create a log
file the CreateFileA function would need to be called. Getting the address of that is a little tricky. Luckily when Windows
boots, all of the Kernel32 functions are loaded at the same location in virtual memory for every process.
Therefore our host program can query the OS where CreateFileA is,
and provide that address to the shell code. The shell code can then cast it to a function pointer with the correct signature
and calling convention, and use the function like it was dynamically loaded.

## Hooking Funtions
Now that the setup is out of the way, we can finally make progress! Because of the complexity introduced by manually
loading the ELF files, I figured it would be easier to hook the functions manually rather than introducing an external hooking library.
This was thankfully relatively simple because of the way some Kernel32 functions are loaded. They have a stub which is a single jump instruction
to the body of the actual function. To hook the function, just move that instruction down and have a single 32-bit relative jump in the original
position so that whenever it is called it will jump instead. Then in my function, I can decide if the original function should be
called by calling the jump that was moved down.

## JOP and ROP
Removing file validation was a challenging problem. I drew inspiration from cslol-manager. Their solution: search the
memory for a specific pattern, an instruction that contains a jump to a pointer that is stored in static memory 
and is called in the validation routine or by one of its subroutines. By changing out this jump to point to some of our code,
we can scan down the stack to find the address that will be returned to by the check. We then swap out this address for our procedure that sets the value
of RAX to 1 (true) and jumps back. This effectively makes it so no matter what the function returns, we will always make it return true,
always passing validation. Note that this would not be so complicated if there were no unchangeable memory protections on League of Legends 
instructions. Instead, we have to use a mix of jump-oriented programming, stack smashing, and return-oriented programming to get the correct result.
I implemented a version that would work with my code but used their code as a reference, which forced me to update my license from GPLv2 to GPLv3.

## Segment Tables
One problem of optimization I ran into early was the time it took to write out temporary files to load into the game. The game files
can be up to a Gigabyte, which for a slow hard drive would take ages to write out. Initially, if I had a file that I wanted to load,
I would have to merge it with the game files, write that out to disk, and then have League of Legends open that file instead
of the file it was supposed to open. This is used by cslol-manager, but for real-time syncing, I needed a better solution.
Only a few entries of the file needed to be changed. If I hook ReadFile, then I can read most of the entries from the
real game files that already exist, and swap out entries that I want to change when it requests them.

## Syncing States
There were many options for when to send files. I decided that sending last second on game load however is the most user-friendly.
There would be no wasted data transfer, and it was now possible because of my segment table optimization.
I was already suspending and resuming threads to set the thread context and run my code, I could suspend the thread,
download the files, and then resume the thread once everything was ready. This made it easy to use a simple request/response architecture.
Web frameworks make using a request/response architecture easy and create the possibility of using a CDN for caching later if
it becomes a problem.

## Optimization
One final thing that had to be changed was the optimization of waiting for a process to start. On
Windows there is no "notify me when a process has been created". Polling was my best option. I had to
poll often (5ms-20ms) because of time sensitivity. If I hook the functions too late everything 
breaks because some functions load the original game data. Therefore this routine needed to be optimized because
it was being called so often and very taxing on the CPU for something that should be idle. My solution
was to have 2 buffers of pointers. EnumProcesses writes to a buffer of process handles, and from experimentation, if nothing has changed
then it will write in the same order. I can hold the last buffer and the new buffer, and to begin we do a super cheap memory comparison.
If there is a  difference, then check a HashSet of the last process handles and for each new one, open it and check the name, if the name matches
the path of League of Legends then load it, if it does not then close the process and add it to the HashSet so it does not
need to be opened again. Overall this saved about 10x performance over just using a HashSet because most of the time no process
is created or deleted.

