Oh Docker. How much I love reading, experimenting and using you.

All the industries that you can think of are moving towards Docker and the container methodology for shipping and running software. I'm glad to be here to write, code and innovate with it, less be along for the exciting ride.

Having an exciting software job at an exciting company that was just acquired is opening up so many possibilities, and making possible the projects that I want to accomplish to make things better. Whether it's big projects like containerizing our multiple apps or implementing Continuous Delivery, or smaller projects like upgrading our build scripts from Ant to Gradle, or updating to the latest Java version (lambda all the things); it's nonstop fun that upgrades my skills, improves the company and gives me a track record.

It's hard to focus on school when all of the brain-twisting fun and innovation is being had over at work. It's a struggle, but a good one since I'm so close to finishing my degree; 1.5 more semesters to be exact. After that, I'm released to continue my fast paced learning concerning whatever I please, backed by an expensive piece of paper. (Maybe a masters degree is to be had further down the line?)

This fourth year Computer Security class that I'm currently taking is immensely fun. I'm glad that I have a super interesting class this semester. I can give credit to the Security Now podcast for which I've listened to around 500 of its 543 episodes as of this writing (read: 10 years of listening!), for giving me the practical knowledge of current security practices and news, diving deep into the details where necessary.

Dr Anil Somayagi, the professor for the Computer Security course, is an excellent lecturer and a hacker at his roots. His interactive teaching style makes the possibly dry subject of security fun and interesting (if you think of security as dry. Who would?), and his course work is very useful in that it promotes self teaching and helping out others. Each week a hacking journal is required for each student to submit. It consists of the student writing about their adventures, frustrations, successes and failures of hacking on security related things - whether that involves using Metasploit to break into a computer, putting a backdoor into some software, figuring out how to configure a firewall, etc. the list goes on and on.

My plan is to work my ass off in all of my classes, finish up my degree and follow my passions, expressing my knowledge and solutions at both my job and in my blog. Ultimately trying to achieve a successful and happy career.

I'm just glad that I don't have a crappy professor.

</rant>


Well, that turned into more of a blog post than anything. Time to start hacking, I suppose.

Now what I'm really talking about with Docker security features isn't SELinux, its seccomp. Seccomp is a kernel module that allows you to put a process into a secure state. In this secure state all system calls are checked if they're allowed in a policy file. If they're not allowed the process gets a SIGKILL signal sent to it from the kernel.

Docker offers containerization of applications (processes) by using cgroups, a Linux technology that limits and isolates resources (memory, cpu, storage, network, etc.). Resources are isolated, but some system calls can still affect the host (eg. clock_settime, mount, reboot).

This is where seccomp comes into play. Docker uses a default policy whitelist of safe system calls that won't affect the host. Users of docker can then override the default policy to use a custom seccomp policy on a per container basis. If a prohibited system call is executed, Docker kills the container by sending a SIGKILL.

Let's create a seccomp whitelist profile for the nginx webserver. I plan on grabbing the nginx docker image, running the nginx server using strace, then grepping the stdout of strace to get all of the syscalls that are needed to run nginx. The syscalls will then be placed into a specially formatted seccomp json file. After all that, I should be able to run nginx using this seccomp policy, preventing any unauthorized system calls to be executed.

The security introduced by seccomp would prevent an attacker who has control over the processes in the container from executing system calls that are prohibited. (eg. The hacker exploits nginx, but can't run programs that use system calls outside of the whitelist).

Alright, lets "docker run nginx". Looks like its running, lets curl its ip: "curl 172.18.0.2". Yep, its up.

Now to run strace to capture the system calls. Lets close down that container and launch a new one, this time overriding the command that is run.


jon@dals-ourt:~$ docker run -it -p 80:80 --name nginx nginx strace nginx "-g daemon off;"
exec: "strace": executable file not found in $PATH
Error response from daemon: Cannot start container f4710a7fd0976b0be297e785c944ef60f444eb0c991eed6af6cdcbe9ad111457: [8] System error: exec: "strace": executable file not found in $PATH

Well that's not good. I'll start a shell in the container, install strace, then try "strace nginx -g daemon off;" again.

Container restarted, strace is installed.

Well look at this:

root@f488df73291c:/# strace nginx -g daemon off;
strace: test_ptrace_setoptions_for_all: PTRACE_TRACEME doesn't work: Permission denied
strace: test_ptrace_setoptions_for_all: unexpected exit status 1

It appears that strace doesn't have the permissions required to run. I read about being able to start a container without any seccomp profile enabled. This should let me run any syscall. I hope this doesn't cause my laptop to crash.

Let's do it for the sake of science! I'll add the following to the "docker run" command: "--security-opt seccomp:unconfined"

Didn't work. Looking at the man pages for docker-run it looks like "--security-opt label:disable" is what I want.

Bingo. Ok, lets see if strace is good.

Ffffff, nope. Same error. After some googling later: god damn apparmor!!!! Checking the message buffer of the kernel (via dmesg) shows that apparmor is denying the strace request. This one issue on Github references an apparmor command to run:

aa-complain /etc/apparmor.d/docker

Okay, this doesn't seem to be working, I'm getting a python stacktrace. Maybe I'll try shutting down the docker service, and read the aa-complain manpages?

Screw this. I'm just going to run the container as privileged and hope it works.

Yeye, it worked. Nginx is running into an error or something. Time to investigate. Looks like wrapping the nginx parameters in double quotes works. It didn't like the multiple spaces

strace nginx "-g daemon off;"

Now that I have successfully captured the system calls during startup, operation and shutdown of nginx, I now have to take the system calls out of the log file and put them into the policy file.

Crap, I missed the "-ff" flag for strace. It tells strace to follow children processes. With nginx, a worker process is spawned after startup. Time to redo it all again.

A quick "docker logs nginx > hack-log-3-strace.log" to output the container's stdout/stderr to a file will now allow me to grep the system calls out.

perl -lne 'print $1 if /([a-zA-Z_]+\()/' "hack-log-3-strace.log" | sort -u

access
arch_prctl
bind
brk
chown
clone
close
connect
epoll_create
epoll_ctl
epoll_wait
execve
exit_group
fcntl
fstat
getdents
geteuid
getrlimit
gettid
htons
ioctl
io_setup
listen
lseek
mkdir
mmap
mprotect
munmap
open
openat
prctl
pread
pwrite
read
recvfrom
recvmsg
rt_sigaction
rt_sigprocmask
rt_sigreturn
rt_sigsuspend
sendmsg
setgid
setgroups
setitimer
set_robust_list
setsockopt
set_tid_address
setuid
shutdown
socket
socketpair
stat
uname
unlink
WIFEXITED
write
writev


Niiiicee, that's a good list of system calls. Let's create the seccomp json file using the format provided in the Docker docs: https://github.com/docker/docker/blob/master/docs/security/seccomp.md

Finished creating the json seccomp policy file. Time to try it out with the nginx container.

So these seccomp profiles are so bleeding edge that its only in the release  candidate of Docker 1.10 at the time of this writing. Going to go get that right now.

Installed the RC build. And it looks like docker doesn't want to run containers anymore. Oh crap I just remembered - this new version has a lengthy migration that commences when first started. Time to monitor "top" and wait for that to finish.

I'm impatient, "iostat" and "iotop" aren't installed on my system, but I can fallback to "sudo lsof | grep docker" to see all the open files. Bingo: Docker's log file is at /var/log/upstart/docker.log. It's doing the migration. I am good.

Well this is going to take forever. I think I'll leave the library and go home. There's three hours of working on this so far.

Sunday...

Tangent: That process took a little over two hours to complete. Maybe I should do some housecleaning sometime and get rid of all those old images I have.

Let's give this a go.

docker run -it -p 80:80 --rm --name nginx --security-opt seccomp:$(pwd)/hack-log-3-profile.json nginx
seccomp: config provided but seccomp not supported
docker: Error response from daemon: Cannot start container d9c162bf7d73bc1c66be696e4190d40d0052ff6f1e96db0440e01eb6fd62da5c: [9] System error: seccomp: config provided but seccomp not supported.

Crap. Gotta figure out why seccomp isn't supported.

A few hours later and I now know that I have to run docker on a kernel that has the CONFIG_SECCOMP flag set when the kernel is compiled. Darn. Well, I could modify and rebuild the kernel for my host machine, or I could fire up a virtual machine and then run Docker in that. Another roadblock. :(

I'd think going the virtual machine route will be better since my host's kernel won't get the slight performance hit, and its my main computer!

Monday...

I'm going to give this two hours of my Library time of creating the VM, then hand this in. What a PITA. I'm thinking of finishing this in the next hacking journal though since I'm so close to completion.

Let's configure and build the kernel in a docker container, create an ubuntu virtual machine, then install that kernel on the ubuntu virtual machine.

Following the general guidelines from https://wiki.ubuntu.com/Kernel/BuildYourOwnKernel , I got the source and build dependencies for the Linux 3.13.0-76-generic kernel. I then made sure the CONFIG_SECCOMP kernel config variable was set. Now lets start the build and play the waiting game.

Alrighty, a few calculus questions worth of time later the build has finished. Now I have a few shiny .deb files with my changes. Time to startup that virtual machine and install the kernel.

Monday night...

It's proving to be hard to find the proper network setup for the virtual machine. How hard does it have to be to copy over some files? Gosh. I'm unfortunately going to cut this here and finish it off in my next hacking journal.

Will seccomp beat the bad guys?!? Tune in next time, same place, same time to see your favourite Linux module serve justice one system call at a time.
