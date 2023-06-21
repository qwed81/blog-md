### FTC

## Introduction
I have been a League of Legends player for years, and a few years ago I came across a program
made by the community that would allow for custom character models to be loaded into the game. This fascinated me
however when I dove into the code it went way over my head. A couple of months ago I decided it would
be fun to try and make my own version that addresses some of the issues I had. After all, what is the point
of having custom character models if your friends can not see them as well? And after deciding to start from scratch
and in Rust, my other objective was to make code that was easier to follow.

## The Plan
Unfortunately it is not as simple as switching out the model files. There is a series of checksum's and signatures
on each of their models. Change out a file the game will not load. So we need to swap the file's that the game loads at run
time. This should be the easy part as we can just change what file is opened when the game makes the system call to open
the file. The verification checks are the hard part as we will have to overwrite the instrucitons that verify it is the
proper file and mantain the proper checksums. Luckily the other project had figured this part out, so some of their ideas
can be used to save a lot of time.

## LOL's Defense Mechanism
League of Legends is a competitive game, therefore it has a high priority to prevent cheaters. As such, I soon realized
how hard this was actually going to be. The first thing I tried was running the code through a disassembler only to find
out that the code was encrypted and gets decrypted at runtime. Welp, time to load up WinDbg then... oh, they also have
debugger checks that will crash the game if there is a debugger attached? I guess this is going to really be a pain.
After watching a couple of game hacking videos
on YouTube, I came across the idea of hooking into Win32 functions that can have their address known by querying windows
with GetProcAddress. However these videos also LoadLibrayA to load a DLL with all of their functionallity. Everything that
seems simple I soon realize the game already has a counter to it. They block loading remote library functionallity as well
as the CreateRemoteThread, something slightly more complicated. Now it's time to get really creative.

## A Solution to Running Code
After a couple days of brain storming, I came up with one of my favorite solutions. Not all functions to
this process are blocked. I am allowed to use WriteProcessMemory, which is the backbone of the idea. However this is
not all I need to get my code to run in their process. I went through the windows documentation and a couple
of functions caught my eye. GetThreadContext and SetThreadContext, functions that are traditionally used for debuggers
allow me set RIP to whatever code position I want. By writing some assembly code that will execute the instructions I want
and then jump back to the value of RIP before I changed it while keeping all registers intact, I can effectively hijack one
of their threads to do the work for me.

## Writing Assembly
The goal is to write a "procedure" that does all of my work and then jumps back to an arbitrary position while maintaining
all of the registers. When starting this project, League of Legends still ran into 32 bit mode, meaning my instruction set
was limited. At first I didn't think it was possible to jump to a variable location without using any of the registers. Then
I relaized this is a perfect time to use self modifying code! By using a 32 bit jump to a relative location, I can do the
math of the offset that is required to jump, and then write that into the actual instruction itself. When the cpu reaches that
instruction, it will execute the jump without looking at any of the registers, so I can set the registers to the state they
were in prior to my initial change of RIP.

## Elegant Shell Code
Traditionally shell code is written in assembly, however I had very complex constraints that I needed to fill. To write it 
all in assembly, while possible would not be optimal usage of time and would become unmanagable. I had to make a decesion
on what language the shell code would be written in, should I keep it all in rust or use another language potentially increasing
the complexity of project. The problem with writing it in rust, is my problem is inherently unsafe. Interfacing with assembly,
calling conventions, pointer arithmatic, casting absolute values to function pointers, and calling windows api functions directly.
For this reason I chose to go with C, a choice that I am super glad I made. But now the problem is how do I get my compiler to
output something that is actually useful to me? How about make it produce a linux ELF file for x64 arcitecture? I mentioned earlier
that I cannot compile to windows files as LoadLibrary is blocked by the game. However because I can WriteProcessMemory, I can
allocate my own segments, parse the linux ELF file, and map the ELF segments from the file into the process memory.
This requires that the output is position independent with the option -fPIE and -fPIC, because I do not know where the process will
be mapped at runtime. Also because we are compiling a linux executable to use on windows the standard library does not make sense to
use. Therefore -nostdlib was also used to ensure no weird items were being linked. The goblin library helped as it provided a high
level API for working with ELF files, allowing me to focus on basic math of mapping to relative addresses, and calling the proper
windows functions to get it in to the League of Legends running process address space. Once the code was compiled and mapped
into the address space of the other process, it became as simple as running specifying the calling convention of the main function,
telling it there were no callee saved registers (makes it easier on the bootstrapping assembly) and then from the assembly calling
the function with the correct parameters.

## Side Effects in Shell Code
It is impossible to know at compile time the address of the windows functions that we need to call. For example to create a log
file the CreateFileA function would need to be called. But to get the address of that is a little tricky. Luckily when windows
boots all of the Kernel32 functions are loaded at the same location, therefore our program can query the OS where CreateFileA
is and then give that address to the shell code. The shell code can then cast it to a function pointer with the correct signature
and calling convention and call the function. It's important to note that now the process calling this function is League of Legends,
so it will retain the cwd of League instead of the other program.

## Hooking Funtions
Now that the setup is out of the way, we can finally make progress! Unfortuanately because of all of the restrictions I figured
it would be easier to hook the functions manually rather than dealing with an external hooking library. This was thankfully
very easy because of the way some Kernel32 functions are setup. They have a stub which is a single jump instruction to the actual
function. Nice! Just move that instruction down and have a single 32 rel jump in the initial position so that whenever it is called
anywhere in the program it will go to our jump instead. Then in our function after doing what we needed to do, we call the instruction
that was moved down (the inital jump) with the new parameters set and optionally return that return value.

## JOP and ROP
Removing file validation was actually a very hard problem and I drew inspiration from the other project. Their solution: search the
memory for a specific pattern. This is an instruction that contains a jump to a pointer that is stored in static memory 
that is called in the validation routine. By changing out this to point to some of our code we can then go and scan down the stack to
find an address that will be returned to by the check. We then swap out this address for our own procedure that simple sets the value
of RAX to 1 (true), and jump back. This effectively makes it so no matter what the function returns, we will always make it return true
so anything passes validation. Note that this would be not be so complicated if there were not memory protections on their instructions
but instead we have to use a mix of jump oriented programming, stack smashing, and return oriented programming to get the correct result.
I implemented my own version that would work better with my code, but used their code as reference, making my code now GPLv3.

## Segment Tables
One problem of optimization I ran into early was the time it took to write out temporary files to load into the game. For example
if I had a file that I wanted to load, first I would have to merge it with the game files, write that out to disk, and then have
League of Legends open that file instead of the file it was supposed to open. However because only a slight amount of the entries
of the file needs to be changed, there is a pretty creative optimization. Hook ReadFile instead. If the requested entry is already
on disk, then it can read from that file. If it is an entry I want to replace, then don't call ReadFile, but instead just replace
the bytes of the buffer with the data I want to replace with.

## Syncing States
There were many options of when to send files. I decided that sending last second on game load however is the most user friendly.
Because I was already suspending and resuming threads to set the thread context and run my code, it made sense to suspend the thread,
download the files, and then resume the thread once everything was ready. This made it easy to use a simple request/response architecture
which made it a perfect canidate to use a web server because most of the foundation is already there.

## Optimization
One final thing that had to be changed was the optimization of loading waiting for a process to start. On
windows there is no "notify me when a process has been created" so polling was my best option. I had to
poll often however because of the time sensitivity of loading, hook the functions too late and every thing 
breaks. This function therefore had to be optimized because it was being called so often. My solution
was to have 2 buffers. EnumProcesses writes to a buffer of process handles. If they are the same as the last call it will most likely
be in the same order (from expermentation). Therefore to begin we do a memory comparison, which is super cheap. If there is a 
difference, then check a HashSet of the last process handles and for each new one, open it and check the name, if the name matches
the path of League of Legends then load the stuff, if it does not then close the process and add it to the HashSet so it does not
need to be opened again. Overall this saved about 10x perfomance over just using a HashSet. I didn't try opening and closing all
the processes, but I assume it would have performed terribly.


