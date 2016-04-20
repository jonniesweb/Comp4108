Paper: Boxify: Full-fledged App Sandboxing for Stock Android

(Sidenote: Whoah, this was a great paper.)


### What is the security problem the paper addresses? In what kind of threat model(s) does the problem exist?
This paper addresses the problem of Android app security. Because users are installing any app from the Android Marketplace, the user doesn't know if the app is malicious or not, but they would like to protect their device and it's information from being compromised. There are other solutions but they're categorized into two camps. One is OS extensions, where the kernel is modified. This makes it hard to deploy, but offers strong security. The other camp of solutions is application layer solutions, where the solution runs as an app. This makes it easy to deploy, but impossible to monitor other apps because of app separation. It would only be useful for monitoring the operating system. Boxify provides a framework to build security policies on, it doesn't actually implement any.

The attacks that Boxify tries to defend against are both an authenticated and unauthenticated local threat. This is local because the untrusted apps are installed and run as a process on the operating system. The code their running is untrusted, but they might be given various permissions to run based on the Android permission scheme.

### How significant is the problem? Specifically, to what degree do existing solutions not work sufficiently well?

The current security problem we're seeing is that the Android system isn't providing enough flexibility with security measures. The user has a very coarse way of allowing permissions for each app. Even then, with the "Access the Internet" Android permission, the app is free to communicate with anything on the internet that it wishes to do. The user has no control over it.

With current solutions, as quickly discussed above, there is a trade-off of convenience of deployment to security. The OS extensions method was hard to deploy but offered good security. The other was application layer solutions, where it's easy to deploy, but doesn't offer as good security. Boxify offers a solution that has the best of both worlds; it offers both strong security and easy deployment by isolating the app and running the reference monitor code in userspace.

### What is the defense? How does it work?

The defense presented by the authors of the paper describe a framework for sandboxing Android apps from the Android operating system. The framework implements a reference monitor for IPC and system calls. It allows for others to write policies that would restrict the effects an app could have on the Android system. Boxify loads an app to be sandboxed into a isolated process, which is a very restricted process provided by the Android system. Boxify then overwrites some segments of the ELF file with the addresses of Boxify's reference monitor. This is how Boxify manages to shim itself in between the app and the operating system.

### To what degree will the defense potentially solve the targeted security problem? In particular, how difficult will it be for attackers to adapt to this defense?

Boxify unfortunately reimplements the Android permission model inside of itself to virtualize the applications that it sandboxes. Because of this, there could be bugs in that code. Since Boxify needs all permissions from the Android system to run, if an app running in Boxify were to break out of the sandbox and take control of Boxify, it would have a far larger stack of allowed permissions to use. Simply put, theres more places that the attackers can pry at.

### What are the challenges facing deployment of the defense? Are they likely to be overcome?

The authors presented a very easy to deploy solution. Their solution consisted of a single app that is installed on the user's Android device. When a user then wants to install an app into Boxify, they can do so from the regular Android Marketplace and the apk file would be installed into the Boxify app.

Some foreseen challenges could be that Boxify won't gain widespread use since it's not clear to the regular non-techsavvy Android user that the security problem potentially fixed by Boxify is great enough to warrant installing. Users could think that their device is secure enough, or that they don't need additional security for what they do. Challenges like this are likely to be overcome if Boxify comes default with Android, or it's bundled into the software provided by a manufacturer who offers a customized Android operating system.

### For both kinds of papers, you should give your reaction by addressing questions like the following:

### Did you like the paper?

I really enjoyed reading this paper because it discussed in depth an ingenious way to secure apps in userspace. It thoroughly reminds me of the work I did in a Hacking Journal on configuring Seccomp in a Docker container, which is basically a reference monitor for system calls made from within a container. I also enjoyed the paper since the results showed a modest overhead in the performance of running sandboxed applications.

Another point that really interested me was the possibility of running ad blockers or Xposed modules in userspace instead of requiring root access. This would most likely future proof both ad blocker and Xposed plugins development.

### Was it easy to understand, or was it hard to read?

I was initially confused, thinking that the paper was providing a secure solution to preventing apps from gaining access to the system and bypassing security measures. I scoffed at the thought that it was a perfect solution, since the authors discussed that they use a system call reference monitor to verify system calls. From a previous Hacking Journal I found out the hard way that it's very hard to create a white list of system calls as there's just too many code paths to exercise to make sure all system calls were picked up. Not too later in the paper I realized that the paper was presenting a framework for implementing these kinds of rules, not implementing these "magical" rules for all Android apps.

### Did you learn much from the paper?

I learned that the Android ecosystem is also going through a containerization and virtualization period, just like the enterprise software ecosystem. Overall, I didn't learn much in the way of the technology that's used to sandbox apps, since sandboxing is the same where ever you look and it's still an issue in many other ecosystems. All of the sandboxing information comes back to the OS reference monitor, and the discussion of a secure operating system. It's the same fundamentals, but applied to a specific ecosystem.

### How surprised were you by the result?

I was really surprised about the minimal performance overheads that were shown in the paper. A 5% overhead can be undetectable to the user and is cost effective considering the added security benefits. Unfortunately the results don't take into consideration the performance overhead introduced by the security policies that will use the framework. The real results can only be higher than those shown in the paper.
