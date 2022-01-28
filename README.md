# SystemD sandboxed Services

Most Linux distro's ship with SystemD these days - which both has a big yay- and nay side.

Sometimes however - especially in professional environments - you might not be able to avoid using SystemD.

The very least you could do in that case is to isolate your services as much as possible. And even if you _do_ like systemd, isolating your services is still a very good practice!



SystemD does offer some nice tools on that, some of which take some trial-and-error to get right, but you'll get there.


## About this project.

This project is distributed as-is. That this works for me doesn't neccisarily mean it will work for you.
The services listed in this repository have been tested to work on the latest _Debian Stable_ (Bullseye).

In example: The DNSMasq service file has been tested in a case where DNSMasq only serves DNS, no DHCP. So it might be possible that SystemD will just kill it as soon as you start doing DHCP..

Pull requests however, are mostly welcome. Please follow the PR-Guidelines you'll see when making a PR to make sure I can properly validate your work.
If successfull your PR will be implemented.


## Installation

The best way to install these unit files is to apply them as a _override_.
On most distro's directly editing the file in `/usr/lib/systemd` or `/etc/systemd` will result in the file being overwritten when the affected service is updated.

This can be done in either of two ways:

__Creating override files__
_This method has been tested on Debian Bullseye, paths may differ on other distro's__

Installation is simple and straightforward. I will once again use dnsmasq as the example service.

1.  Clone the repo `git clone https://github.com/justSem/systemd-sandboxed-services.git`
2.  Copy the required directories from the `services` directory. `cp -r systemd-sandboxed-services/services/dnsmasq.service.d /etc/systemd/system`
3.  Execute `systemctl daemon-reload` to make SystemD integrate the changes
4.  When you run `systemctl status dnsmasq` you'll see `(Sandbox override file loaded)` appended to the description.
5.  Don't forget to restart the affected service for the changes to take effect `systemctl restart dnsmasq`


__Using systemctl edit__

By using the `systemctl edit <unit>` command you'll get an editor where you can paste the edited service file.
This method is the most automated means possible as systemd will take care of the rest. This will be labor-intensive though.
_Note: You'll still need to restart the affected services__


## Warning

If you decide to start tinkering around yourself, please be especially careful around the `SystemCallFilter=` expressions.

Unlike `CapabilityBoundingSet=` which just causes your service to not start if you misconfigure it, the SystemCallFilter will make Systemd _kill_ your service as soon as it get's triggered.

A simple example: Let's say your service doesn't require any filesystem operations until you decide to interact with it - but you don't have `@basic-io @file-system` in your SystemCallFilter whitelist. Then interacting with said service might cause it's first I/O event and thus trigger the SystemCallFilter - resulting in SystemD immediately killing the service.


## Tips and recommendations.

An easy way to asses how 'open' your services are is by running `systemd-analyze security`

An example output might look like this:

```
sem@majora:~$ systemd-analyze security
UNIT                                 EXPOSURE PREDICATE HAPPY
auditd.service                            8.8 EXPOSED   🙁
bird.service                              9.6 UNSAFE    😨
bird6.service                             9.6 UNSAFE    😨
cron.service                              9.6 UNSAFE    😨
dbus.service                              9.6 UNSAFE    😨
dnsmasq-pub.service                       9.6 UNSAFE    😨
dnsmasq.service                           2.3 OK        🙂
e2guardian.service                        9.6 UNSAFE    😨
emergency.service                         9.5 UNSAFE    😨
getty@tty1.service                        9.6 UNSAFE    😨
ifup@ens3.service                         9.5 UNSAFE    😨
nginx.service                             9.6 UNSAFE    😨
packagekit.service                        9.6 UNSAFE    😨
polkit.service                            9.6 UNSAFE    😨
rc-local.service                          9.6 UNSAFE    😨
rescue.service                            9.5 UNSAFE    😨
rsync.service                             8.5 EXPOSED   🙁
rsyslog.service                           9.6 UNSAFE    😨
sdwdate-aae.service                       8.3 EXPOSED   🙁
ssh.service                               9.6 UNSAFE    😨
systemd-ask-password-console.service      9.4 UNSAFE    😨
systemd-ask-password-wall.service         9.4 UNSAFE    😨
systemd-fsckd.service                     9.5 UNSAFE    😨
systemd-initctl.service                   9.4 UNSAFE    😨
systemd-journald.service                  4.3 OK        🙂
systemd-logind.service                    2.6 OK        🙂
systemd-networkd.service                  2.9 OK        🙂
systemd-rfkill.service                    9.4 UNSAFE    😨
systemd-timesyncd.service                 2.1 OK        🙂
systemd-udevd.service                     8.0 EXPOSED   🙁
tor@default.service                       6.6 MEDIUM    😐
unattended-upgrades.service               9.6 UNSAFE    😨
unbound-resolvconf.service                1.8 OK        🙂
unbound.service                           3.9 OK        🙂
user@0.service                            9.8 UNSAFE    😨
wazuh-agent.service                       9.6 UNSAFE    😨
```

This shows you that SystemD is happy about a few services, but mostly unhappy about others.

To display further details about a service you could run `systemd-analyze security <unit>` In the next example we'll take a look at DNSMasq


```
systemd-analyze security dnsmasq
  NAME                                                        DESCRIPTION                                                                                     EXPOSURE
✗ PrivateNetwork=                                             Service has access to the host's network                                                             0.5
✗ User=/DynamicUser=                                          Service runs as root user                                                                            0.4
✗ CapabilityBoundingSet=~CAP_SET(UID|GID|PCAP)                Service may change UID/GID identities/capabilities                                                   0.3
✓ CapabilityBoundingSet=~CAP_SYS_ADMIN                        Service has no administrator privileges
✓ CapabilityBoundingSet=~CAP_SYS_PTRACE                       Service has no ptrace() debugging abilities
✗ RestrictAddressFamilies=~AF_(INET|INET6)                    Service may allocate Internet sockets                                                                0.3
✓ RestrictNamespaces=~CLONE_NEWUSER                           Service cannot create user namespaces
✓ RestrictAddressFamilies=~…                                  Service cannot allocate exotic sockets
✓ CapabilityBoundingSet=~CAP_(CHOWN|FSETID|SETFCAP)           Service cannot change file ownership/access mode/capabilities
✗ CapabilityBoundingSet=~CAP_(DAC_*|FOWNER|IPC_OWNER)         Service may override UNIX file/IPC permission checks                                                 0.2
✓ CapabilityBoundingSet=~CAP_NET_ADMIN                        Service has no network configuration privileges
✓ CapabilityBoundingSet=~CAP_SYS_MODULE                       Service cannot load kernel modules
✓ CapabilityBoundingSet=~CAP_SYS_RAWIO                        Service has no raw I/O access
✓ CapabilityBoundingSet=~CAP_SYS_TIME                         Service processes cannot change the system clock
✗ DeviceAllow=                                                Service has a device ACL with some special devices                                                   0.1
✗ IPAddressDeny=                                              Service does not define an IP address allow list                                                     0.2
✓ KeyringMode=                                                Service doesn't share key material with other services
✓ NoNewPrivileges=                                            Service processes cannot acquire new privileges
✓ NotifyAccess=                                               Service child processes cannot alter service state
✓ PrivateDevices=                                             Service has no access to hardware devices
✓ PrivateMounts=                                              Service cannot install system mounts
✓ PrivateTmp=                                                 Service has no access to other software's temporary files
✗ PrivateUsers=                                               Service has access to other users                                                                    0.2
✓ ProtectClock=                                               Service cannot write to the hardware clock or system clock
✓ ProtectControlGroups=                                       Service cannot modify the control group file system
✓ ProtectHome=                                                Service has no access to home directories
✓ ProtectKernelLogs=                                          Service cannot read from or write to the kernel log ring buffer
✓ ProtectKernelModules=                                       Service cannot load or read kernel modules
✓ ProtectKernelTunables=                                      Service cannot alter kernel tunables (/proc/sys, …)
✓ ProtectProc=                                                Service has restricted access to process tree (/proc hidepid=)
✓ ProtectSystem=                                              Service has strict read-only access to the OS file hierarchy
✓ RestrictAddressFamilies=~AF_PACKET                          Service cannot allocate packet sockets
✓ RestrictSUIDSGID=                                           SUID/SGID file creation by service is restricted
✓ SystemCallArchitectures=                                    Service may execute system calls only with native ABI
✓ SystemCallFilter=~@clock                                    System call allow list defined for service, and @clock is not included
✓ SystemCallFilter=~@debug                                    System call allow list defined for service, and @debug is not included
✓ SystemCallFilter=~@module                                   System call allow list defined for service, and @module is not included
✓ SystemCallFilter=~@mount                                    System call allow list defined for service, and @mount is not included
✓ SystemCallFilter=~@raw-io                                   System call allow list defined for service, and @raw-io is not included
✓ SystemCallFilter=~@reboot                                   System call allow list defined for service, and @reboot is not included
✓ SystemCallFilter=~@swap                                     System call allow list defined for service, and @swap is not included
✗ SystemCallFilter=~@privileged                               System call allow list defined for service, and @privileged is included (e.g. chown is allowed)      0.2
✓ SystemCallFilter=~@resources                                System call allow list defined for service, and @resources is not included
✓ AmbientCapabilities=                                        Service process does not receive ambient capabilities
✓ CapabilityBoundingSet=~CAP_AUDIT_*                          Service has no audit subsystem access
✓ CapabilityBoundingSet=~CAP_KILL                             Service cannot send UNIX signals to arbitrary processes
✓ CapabilityBoundingSet=~CAP_MKNOD                            Service cannot create device nodes
✗ CapabilityBoundingSet=~CAP_NET_(BIND_SERVICE|BROADCAST|RAW) Service has elevated networking privileges                                                           0.1
✓ CapabilityBoundingSet=~CAP_SYSLOG                           Service has no access to kernel logging
✓ CapabilityBoundingSet=~CAP_SYS_(NICE|RESOURCE)              Service has no privileges to change resource use parameters
✓ RestrictNamespaces=~CLONE_NEWCGROUP                         Service cannot create cgroup namespaces
✓ RestrictNamespaces=~CLONE_NEWIPC                            Service cannot create IPC namespaces
✓ RestrictNamespaces=~CLONE_NEWNET                            Service cannot create network namespaces
✓ RestrictNamespaces=~CLONE_NEWNS                             Service cannot create file system namespaces
✓ RestrictNamespaces=~CLONE_NEWPID                            Service cannot create process namespaces
✓ RestrictRealtime=                                           Service realtime scheduling access is restricted
✓ SystemCallFilter=~@cpu-emulation                            System call allow list defined for service, and @cpu-emulation is not included
✓ SystemCallFilter=~@obsolete                                 System call allow list defined for service, and @obsolete is not included
✗ RestrictAddressFamilies=~AF_NETLINK                         Service may allocate netlink sockets                                                                 0.1
✗ RootDirectory=/RootImage=                                   Service runs within the host's root directory                                                        0.1
  SupplementaryGroups=                                        Service runs as root, option does not matter
✓ CapabilityBoundingSet=~CAP_MAC_*                            Service cannot adjust SMACK MAC
✓ CapabilityBoundingSet=~CAP_SYS_BOOT                         Service cannot issue reboot()
✓ Delegate=                                                   Service does not maintain its own delegated control group subtree
✓ LockPersonality=                                            Service cannot change ABI personality
✓ MemoryDenyWriteExecute=                                     Service cannot create writable executable memory mappings
  RemoveIPC=                                                  Service runs as root, option does not apply
✓ RestrictNamespaces=~CLONE_NEWUTS                            Service cannot create hostname namespaces
✓ UMask=                                                      Files created by service are accessible only by service's own user by default
✓ CapabilityBoundingSet=~CAP_LINUX_IMMUTABLE                  Service cannot mark files immutable
✓ CapabilityBoundingSet=~CAP_IPC_LOCK                         Service cannot lock memory into RAM
✓ CapabilityBoundingSet=~CAP_SYS_CHROOT                       Service cannot issue chroot()
✓ ProtectHostname=                                            Service cannot change system host/domainname
✓ CapabilityBoundingSet=~CAP_BLOCK_SUSPEND                    Service cannot establish wake locks
✓ CapabilityBoundingSet=~CAP_LEASE                            Service cannot create file leases
✓ CapabilityBoundingSet=~CAP_SYS_PACCT                        Service cannot use acct()
✓ CapabilityBoundingSet=~CAP_SYS_TTY_CONFIG                   Service cannot issue vhangup()
✓ CapabilityBoundingSet=~CAP_WAKE_ALARM                       Service cannot program timers that wake up the system
✗ RestrictAddressFamilies=~AF_UNIX                            Service may allocate local sockets                                                                   0.1
✓ ProcSubset=                                                 Service has no access to non-process /proc files (/proc subset=)

→ Overall exposure level for dnsmasq.service: 2.3 OK 🙂
```

This overview will give you a list of all SystemD's sandboxing options and will show you how it affects the current service.
As you can see here SystemD is pretty content and there isn't a lot to be done.

I often use this tool when making new files to keep a general overview.
