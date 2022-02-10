> NOTICE: This guide is under construction and will be finished in the coming few days. Please check back later if you want the full read. (10-Feb-2022)

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
- Nginx uses ports < 1024. By default this requires the `NET_BIND` capability (more on capabilities later).
- Nginx can do - in special setups - some freaky stuff with sockets and all kind of TCP connections, which might require the spawning of `AF_UNIX` or `AF_NETLINK` sockets.
