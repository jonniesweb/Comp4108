Feb 2, 2016

Let's finish off what I started in the third hack log - installing a kernel... actually, that's only part of it. The main goal is to get seccomp to secure a Docker container.

Alrighty, I have an existing Xubuntu virtual machine. It took me a while to figure out the convoluted method for installing the VirtualBox guest additions and picking the right bridged network to connect to the VM. Now I'm able to ssh into the host machine from the VM, ultimately allowing for sftp access.

Time to sftp over the kernel .deb files and install them using dpkg. (Sometimes you love and sometimes you hate package managers eh?)

Crap.

This VM I had lying around is i386 based. I need amd64. Time to go install a new system. Fml.

Here goes another hour of waiting for stuff.

Big mistake, I shouldn't have selected a desktop manager during install. This *is* taking forever. Don't do a netboot either.

Okay, who knows how long that took, but It's finished installing and I'm able to login now. I sftp'ed the files over to this guest OS and ran "dpkg -i *.deb" to install all those kernel deb files.

Docker is now complaining that seccomp isn't installed. I think I have to go install the seccomp libs. Time to look at the Dockerfile that builds docker. I remember there being commands for installing seccomp from source. Here we go, fetch the source, make, make install, lalala... building all these things is getting repetitive.

Running Docker in Docker in Docker. It's containers all the way down!

It's proving to be very hard to get the docker daemon running in the container, but I finally figured it out. Now trying to run a container in that container is proving to be difficult. When I do "docker run -it nginx" it should fetch the nginx image and run it. But it's freezing halfway through. Maybe I should checkout the tagged RC instead of running off of master.

Aha! Something's not working in master. Switching to the RC tag worked!

Running the nginx container works. Now I just have to assign the security profile.

docker run -it -p 80:80 --rm --name nginx --security-opt seccomp:$(pwd)/hack-log-3-profile.json nginx

Running this command worked. In a way. Seccomp is working, but looks like I have to update the seccomp policy file since It's giving me runtime errors about syscalls being needed.

I think I have to include system calls that are required by a part of docker. It seems like the container isn't even being started. Looking at the github issues and doing some googling has led me to believe that it's currently a bug, or I just need to have a less restrictive seccomp profile.

This is taking a lot of time. I'm going to take a break.



Sunday...

Back at it and I've updated the seccomp profile. I've added a lot of the safe syscalls listed in the recommended seccomp profile from https://github.com/docker/docker/blob/master/profiles/seccomp/seccomp_default.go . Running the nginx container works! Hallelujah!

Learning from this whole process, its apparent that setting up a seccomp profile for Docker is a non-trivial process at the moment. It's extremely hard since there could be code paths that haven't been exercised during analysis, thereby some system calls could be missed.

I see two methods of finding the set of system calls that are required to run the application under test. The first method is performing static code analysis on the program's binary. This becomes more complicated when dynamic libraries are used since the external libraries aren't included with the program. This method would return exact results for the set of required system calls since you're able to parse the complete codebase.

The second method involves dynamic analysis of the program under test. The program is started and used, exercising as many interactions as possible to black-box test the program. At the same time a system call analyzer program, such as strace, would be used to log all of the system calls made by the program. This method is tedious, but can be successful if the exercising of the program is thorough enough to capture all system calls.

In practice, the latter is only used possibly because it's the only one available at the moment that works for both static and dynamic languages. Even when thoroughly exercising the program, it's possible to miss a few system calls. Finding the last few is a non trivial problem at the moment.

Security is hard.
