> NOTICE: This guide is under construction and will be finished in the coming few days. Please check back later if you want the full read. (6th of April 2022)

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
The more capabilities you revoke, the more it'll become a trial-and-error thing. Unless you know in-depth what your service does and doesn't use of course ;)

These kind of capabilities are an example of 'permissions' you can control with Systemd.

### Namespaces
In the rest of this article there will be plenty of mentions on Namespaces.
I recommend you to read a bit about what a namespace is.
Basically, a namespace allows systemd in this case to isolate a process in it's own 'bubble'. This could be a filesystem namespace, but network namespaces might be used as well.
Namespaces can be used for both isolation and functionality.
A big user of network namespaces is docker, which uses the function to stop containers from talking to eachother when they're not supposed to.
Another example of network namespaces is a scenario in which a user utilizes a VPN, but only wants certain services to use that VPN. In that case he can install the VPN adapter in a separate network namespace, which he may assign services to.

__Further reading__
[Linux Namespaces - RedHat](https://www.redhat.com/sysadmin/7-linux-namespaces)
[Linux Namespaces and container technology - Medium](https://medium.com/geekculture/linux-namespaces-container-technology-a09da0813247)

## Sanboxing features
> Note: All of the following features can be found as well in the [systemd manual](https://www.freedesktop.org/software/systemd/man/systemd.exec.html)

I'm going to walk you through most of the sandboxing features Systemd offers. Some are rarely used or just not applicable and are therefore omitted. You can - of course - refer to the man page if you want to know more about them.

__ProtectSystem__
`ProtectSystem` accepts the values: `true`, `false`, `strict` or `full`.
Simply put: this option allows you to make parts of your filesystem read-only with a single command, generally this does the following
| Value | Action |
| -- | -- |
| `false` | No changes |
| `true`  | Mounts `/usr`, `/boot` and `/efi` (if applicable) as read-only |
| `full`  | The same as `true` but adds `/etc` to the list |
| `strict` | Mounts the whole FS as read-only except for the API subtrees (`/proc`, `/sys` and `/dev`) |

There are other options available to protect the paths that aren't covered by `ProtectSystem`. Additionally, you might use `ReadPaths` or `ReadWritePaths` to further fine-tune FS access.


__ProtectHome__
`ProtectHome` accepts the values: `true`, `false`, `read-only` or `tmpfs`
This option is redundant and also covered by others when `read-only` or `tmpf` is used. If `true`, this mounts `/home`, `/root` and `/run/user` as empty, inaccessible directories.


__ReadWritePaths, ReadOnlyPaths, InaccessiblePaths, ExecPaths, NoExecPaths__
The names of these options generally imply their workings.
Paths may be prepended by `-` in which case they'll be ignored if they don't exist (if not, the unit will fail to start). In case a path is prepended with `+` the path will be treated relative to the services' root directory.
You may combine `+` and `-` as `-+`.
Please note that the above options are recusive _by default_.

One may nest `ReadWritePaths` inside of `ReadOnlyPaths` to allow write access in subdirectories of `ReadOnlyPaths`. The same goes for nesting `ExecPaths` in `NoExecPaths`.
You can _not_ nest `ReadWritePaths` or `ReadOnlyPaths` within `InaccessiblePaths`. In case you want a structure similar to embedding accessible paths within `InaccessiblePaths` please see `TemporaryFileSystem`.

When using `ProtectSystem=strict` one may use `ReadWritePaths` to make paths within the filesystem writable for the service.
Keep in mind that for these options to be effective, you'll need to combine them with at least: `CapabilityBoundingset=~CAP_SYS_ADMIN` or `SystemCallFilter=~@mount` so the process can't circumvent the restrictions you've imposed. 


__TemporaryFileSystem__
Use `TemporaryFileSystem` to setup temporary file-system namespaces for the process.
The option may be called more then once, in which case all previous calls are mounted as well.
Calling the option as `TemporaryFileSystem=` will result in the list to be reset, and all previous tmpfs mounts to be dropped.

You may specify other options with your mount points, prefixed by a colon `:`. In example: `TemporaryFileSystem=/var:ro` will mount /var as tmps and as read-only. Generally, most common mount options are accepted.
This option may be used in a nesting-situation where one may for example do the following:
```
TemporaryFileSystem=/var:ro
BindReadOnlyPaths=/var/lib/systemd

```
This results in the process being unable to see anything in /var, except for /var/lib/systemd


__PrivateTmp__
This option is useful if you want a process to be able to write to standard temporary file folders, but also be unable to access anything in those folders that doesn't belong to said process.
When `PrivateTmp` is set to `true`, systemd will mount a new filesystem namespace as /tmp and /usr/tmp which is to be used exclusively by processes under that systemd unit.
Temporary files belonging to this service are deleted once the service is stopped.

It is possible to create a situation where two separate systemd units may access the same `PrivateTmp` 'namespace' by using the `JoinsNamespaceOf` directive.


__PrivateDevices__
When set to true, this option blocks access to most physical devices listed under /dev.
This option sets up a new namespace as /dev, in which a process can only access pseudo-devices (such as /dev/null, /dev/zero) and the TTY subsystem (/dev/ttyXX).
When set to true, this automatically revokes the _capabilities_: `CAP_SYS_RAWIO` and `CAP_MKNOD`. It also installs a system call filter blocking calls grouped under `@raw-io`.
This option also implies: `DevicePolicy=closed`
Also: In case the service is running either as non-root, or without the `SYS_CAP_ADMIN` capability, the option `NoNewPrivileges=yes` is implied.


__PrivateNetwork__
When this option is set to `true`, and no other option is given, this essentially disables network access for a service, except on the loopback interface.
This option mounts the service in a new network namespace, in which only a loopback interface is added.
> Keep in mind, that due to the nature of netns, the `lo` device is not the same across different network namespaced. If you're running a service that's accessible on 127.0.0.1, then you won't be able to access it with the service mounted with this option.

Additionally this option might be used in a scenario where a service is routed over a VPN (for example) in which the VPN adapter can be mounted in the same namespace by chaining systemd-units using the `Requires`, `After` and `JoinsNamespaceOf` directives.
A user may also specify a hard path to a certain network namespace using `NetworkNamespacePath=`. Keep in mind that for `NetworkNamespacePath` to work, the namespace has exist at the moment the service is forked.


__PrivateUsers__
This is useful if you desire to securely detach the service from the user/group database.
Effectively this bars a service from learning about other users on the system, and it creates mappings to achieve that.

To achieve this, systemd sets up a new user namespace for the processes and configures a minimal user and group mapping which maps the `root` user and group as well as the service's own user and group to themselves and everything else to the "nobody" user and group. 
From the service's perspective this means that everything owned by itself and root will be visible as owned by those users, but everything else will be mapped as `nobody`. Hence, effectively isolating the service from learning user/group names and IDs.


__ProtectHostname__
When set to true, this prevents the system's hostname from being changes by the service.
Due to the way this is set-up, this also prevents the service from noticing once a hostname has changed. Meaning you'll need to restart the service in order for it to recognise a changed hostname.


__ProtectClock__
When set to true, this denies writes to both the system and hardware clock by the service.


__ProtectKernelTunables__
This makes all kernel tuneables (of which most only need to be set at boot) accessible through `/proc/sys/`, `/sys/`, `/proc/sysrq-trigger`, `/proc/latency_stats`, `/proc/acpi`, `/proc/timer_stats`, `/proc/fs` and `/proc/irq` as read-only.
This setting is recommended for most services.


__ProtectKernelModules__
When true, disables the loading or unloading of kernel modules.
This setting is recommended for services that don't need extra kernel modules to work.


__ProtectKernelLogs__
Disables both read and write access to the kernel log ring.


__ProtectControlGroups__
Mounts `/sys/fs/cgroup/` as read-only.
This is recommended for most services as virtually no service (with the exception for most container-manager services) requires access to these paths.


__RestrictAddressFamilies__
This option allows you to restrict the kinds of sockets a service may bind themselves to.
You can set this to `none` to completely disable access to the `socket()` system call. You can use a space-separated allow list (i.e. `AF_INET AF_INET6`) of address families to restrict specific address families.
Prefix your list with `~` to use it as deny- rather then a allow-list.
Please note that this only affects sockets launched through the `socket()` system call, other means of accessing sockets (i.e. through [systemd sockets](https://www.freedesktop.org/software/systemd/man/systemd.socket.html#) or through the `socketpair()` system call) are still possible.


__RestrictFileSystems__
Allows you to specify a set of space-separated filesystems a service can open files on.
i.e.: `RestrictFileSystems=ext4 tmpfs` allows access to `ext4` and `tmpfs` filesystems, but denies access to others.


__LockPersonality__
When set to true, this locks down the [personality(2)](http://man7.org/linux/man-pages/man2/personality.2.html) system call, preventing the service from changing execution domains or a personality set through the `Personality=` directive.


__MemoryDenyWriteExecute__
If set to true, attempts to create memory mappings that are writable and executable at the same time, or to change existing memory mappings to become executable, or mapping shared memory segments as executable are denied.
This installs a system-call filter, blocking calls from `mmap(2)`, `mprotect(2)`, `pkey_protect(2)` and `shmat(2)` in which the calls attempt to map memory as both writeable and executable.


__RestrictRealtime__
If set to true, this option effectively blocks access to real-time process scheduling. This may be used in a situation where a process can attempt to use too much cpu time. Which in return could cause Denial-of-Service situations.


__RestrictSUIDSGID__
If set to true, blocks the setting of SUID or GUID bits on files/folders/processes. 
This option is automatically implied if you're using the `DynamicUser` directive.


__PrivateMounts__
This option is a one-way block in which systemd sets-up a new namespace for mounts created by the process.
Effectively this means that a host cannot see mounts created by the service, but the service can see mounts created by the host.


## System call filtering
> To be continued :)
