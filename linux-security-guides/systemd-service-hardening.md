> NOTICE: This guide is under construction and will be finished in the coming few days. Please check back later if you want the full read. (16-Feb-2022)

# Systemd Service Hardening


## 1. Why?

A lot of us host services on Linux servers and most distributions nowadays come with Systemd.
The problem is that most services have _a lot_ of permissions __by default__.

Though there are pro's and cons to systemd it is the most widespread init manager at the time of writing. 
As such, systemd does offer us some nice features to sandbox our services.


## 2. What will sandboxing do to my services?

The answer is: That depends. There is a range of options you can configure of which some will work and some will break.
To fully optimise your sanboxing you need a deeper understanding of what capabilities, system calls, etc. you service needs.

That's beyond the scope of this piece though.

In general the idea of sandboxing in this context is to isolate the services you host as much as possible from the system without breaking them.
Additional features such as `CapabilityBoundingSet` and `SystemCallFilter` will even limit what your service is _allowed_ to do.


## 3. Why should I sanbox my services?

A lot of services are by default ran as root, or have a need for some (subset of) elevated privileges.
This exposes a service to a lot of system data and resources, as you know root can do anything, and so could your service - potentionally.

A few examples of what a random service ran as root _could_ do:
- Reboot the system
- Load arbitrary kernel modules
- Modify running tasks
- Dump some random proces' memory
- And much, much more.


# Getting started


### To be root, or not to be root.
An intial question to ask yourself is: Does my service run as root? And if so, does it need to?

I've seen many examples or services that run as root just because the developer doesn't know how to properly set permissions.
This can't be fully blamed on the developer, after all they're a developer, not a sysadmin. (And I'm not going to point to any project as examples, because we're not going to shame people here :) )

On the other hand there _are_ developers that package their services as rootless or instruct us on how to run them as a normal user.

So how do you determine if your service needs to run as root? Some very popular ones - such as nginx, or php-fpm - run their master process as root, but why?
Well there are a few reasons for that, here are some:
- Nginx uses ports < 1024. By default this requires the `NET_BIND` capability (more on capabilities later). A capability like this can only be assigned to a privileged process, or a process requesting said privileged (which require `execve` or similar).
- Nginx can do - in special setups - some freaky stuff with sockets and all kind of TCP connections, which might require the spawning of `AF_UNIX` or `AF_NETLINK` sockets.

So while there are workarounds to run nginx completely rootless it's massively unpractical and easy to break, and obviously beyond the scope of this topic.


### What do I need to sandbox?
Simple, as much as possible of course!
Some services - especially when they start requiring device access, or do advanced things - might not be that sandbox-able at all.
So in my opinion the main idea is to see how strict we can make our sandboxing without breaking (too much) functionality.

Of course you can try to limit a service's functionality if you know you won't need parts of it. In the nginx example you could try to revoke the AF_UNIX or AF_NETLINK socket permissions.
If that works is dependent on how the program request those rights.
If the program checks those permissions on launch it may break, if it checks them only if you configure the program to use them, you might be in luck.


### Capabilities
There are a few key concepts one needs to understand here - one part of this are [Linux Capabilities](https://man7.org/linux/man-pages/man7/capabilities.7.html).
Simply put: Linux capabilities can be compared to a set of admin roles on, let's say a discord server.

There are (at the time of writing) a total of 41 capabilities, some of which you'll probably never use.

A few examples of these capabilites are:
- *CAP_SYS_BOOT* Allows the program to reboot the system
- *CAP_SYS_MODULE* Allows the program to load/unload kernel modules
- *CAP_NET_BIND_SERVICE* Allows the program to use ports < 1024.

Nginx, our example service, obviously doesn't require permissions to reboot the system, or to load/unload kernel modules.
These are capabilities we can easily revoke, so in case you nginx process might become compromised and starts to run arbitrary code it won't be able to 'just' mess with your kernel.



## Sanboxing features
> Note: All of the following features can be found as well in the [systemd manual](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)

I'm going to walk you through most of the sandboxing features Systemd offers. Some are rarely used or just not applicable and are therefore omitted. You can obviously refer to the man page if you want to know more about them.

> NOTE: to be continued ;)
