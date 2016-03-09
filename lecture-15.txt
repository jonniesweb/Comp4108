Class notes

PAX ASLR Design
- What is pax?
  - patches put together to harden linux
  - Mainline kernel has adopted ASLR mechanism

- Pool of entropy is used to decide where to load sections of executable
- Executables are in the ELF format, consisting of different sections. Each section can be loaded into different areas of memory
- for Intel x86 you get 16 bits of entropy for ASLR
  - Its not enough since it doesn't take long to try 32,000 tries
- 24 bits of ASLR entropy is 16 million
  - still possible to brute force

- When any of the three delta (randomization) variables are leaked/stolen it's trivial for the attacker to know the addresses of everything

- In the paper the authors use a return-to-libc attack, bypassing the stack protection
  - uses return addresses on the stack
  - any process on unix uses the libc library. ie. the functions are always loaded into every program
  - each executable contains the system() system call
    - allows the program to start a shell

- What if the defender turns off execution on the stack
  - This method is called Write or Execute pages (W or X)
  - return-to-libc bypasses the memory protection from W or X

- The problem introduced in the paper is how can they find the libc library when ASLR and W or X is used
  - This is a derandomization attack
  - Try jumping to addresses at random
  - If the address to jump to was invalid, a segfault occurs
  - The authors assume that the service starts a new process for each request (Default Apache webserver behaviour)
    - Its costly to spawn new processes to do work, but it's more secure since it's an entirely separate memory space
    - In cases where the service doesn't fork requests to a new process, it's common for the memory to be rerandomized after a segfault

- the attackers want to find the delta_mmap variable
  - It is the randomization code
    - keys to the kingdom

- It's important to understand the assumptions that the paper makes when doing these attacks
- Assumptions can make the attack much easier

- 64 bit systems running 64 bit processes has a huge memory space that allows for much more randomization of memory addresses
  - at least 40 bits of entropy
  - Increases the number of tries

- Server's crashing all the time should be a big notice that there's an issue
  - ie. an attacker
  - investigating in the Apache webserver attack
    - block the ip address that's causing the process to crash
    - See the data of the incoming request to determine if it's malicious or not
  - pausing the process causes a DoS for the users of the service



Paper: The Geometry of Innocent Flesh on the Bone: Return-into-libc without Function Calls(on the x86)

- shows a new way of organizing return-into-libc exploits
- static analysis can be used to find code sequences for the return-into-libc attack
- basic idea: put a return address on the stack - but not the return address of a function - an arbitrary instruction that's near the end of the function
  - runs the last few instructions of the function, then it does a return
  - that return address is then another jump into arbitrary instructions, repeating the above

- This allows you to assemble arbitrary functions
- static analysis allows you to use libraries other than libc
- with enough functions, you're able to build any sort of exploit
- each sequence of useful code from the end of functions are called "gadgets"

- How are sequences chained together?
  - you push many return addresses and all the needed parameters
  - need a lot of bytes to perform this type of attack

- Once you figure out where the library is randomly loaded, its trivial to jump to any function in the library



What have we learned about buffer overflows? (attack and defence)
- What lessons have we learned?
  - Don't use unsafe languages (ie. C, C++)
  - attacks are very specific, but potentially robust
  - Side: Pwn2Own - if you compromise the machine/device, you take it home or get a reward
    - Attackers chain together many, many techniques to pwn devices
    - This stuff is real, but it is not easy. Depends on lots of little tricks
- What do you require to do a memory corruption attack?
  - access to the same system
  - same programs, patches, versions
  - All of these are required because memory corruption attacks are fragile and very platform specific
- How do you defeat these in general?
  - software distribution
    - all the code that we put out there are the same - we all run the same code/binaries
    - if attacker has the same binaries, they can study them and develop attacks
- Its silly that we can do crazy things to binaries (ie. crash them 30,000 times) without the binary doing anything
