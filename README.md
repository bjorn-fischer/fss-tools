# fss-tools
Tools and scripts for deploying the Linux fair share scheduler on
multi-user systems, scheduling users against each other instead of processes.
This covers the deployment on [ThinLinc](https://www.cendio.com/ "ThinLinc by Cendio") agent hosts. Other scenarios may apply.

## Motivation

On our site we are currently experimenting with fair share scheduling
on ThinLinc agent hosts. At the Center for Biotechnology (CeBiTec) at Bielefeld University
we provide scientist's desktops (internet and office applications,
and scientific applications) for researchers and teachers.

### Use Case

Formerly based on SunRay technology (using Solaris' FSS), we now use a ThinLinc-4.9 setup
with 5 VSM agents (each 56 Cores, 376GB RAM). Our approach uses
Ubuntu-18.04 on the bare metal and each VSM agent is running inside
an LXD-3.0 container, also with an Ubuntu-18.04 user space. This
setup works exceedingly well wrt migration and deployment of
agent hosts.

Usually, about 30 users share a single VSM agent host which handles
the load quite well -- as long as there is no firefox, thunderbird,
or similar process going berserk. It may also occur that an interactive
science app suddenly generates lots of load. These situations lead
to lags for all users on the system deteriorating the general
user experience.

### Solution: Fair Share Scheduler

In order to fix that problem we are experimenting with the fair share
scheduling feature of the Linux kernel which is controlled by the
cgoups API. With fair share scheduling enabled, a user can aquire all
CPU resources he/she wants as long as there is idle time on the
system. When CPU time is exhausted the scheduler splits CPU time
equally across all demanding users. In other words the user cannot
single-handedly slow down the whole system.

> *On multi-user systems, schedule users against each other, not processes.*

### Deploying cgroups

In order to deploy this feature, a cgroup `cpu:/user/<username>` has to be
created and all processes of each user has to be assigned to the
users cgroup. Then fair share scheduling can be activated by
setting an abstract CPU share weight for each user cgroup:

    root# cd /sys/fs/cgroup/cpu,cpuacct
    root# mkdir -p user/usera
    root# for pid in `pgrep -u usera`; do echo $pid > user/usera/tasks; done
    root# echo 512 > user/usera/cpu.shares

Alternatively, one can use the cgtools instead of fiddling with
`/sys/fs/cgroup` directly:

    root# cgcreate -g cpu:/user/usera
    root# cgset -r cpu.shares=512 /user/usera
    root# cgclassify -g cpu:/user/usera `pgrep -u usera`

The procedure is necessary for all users on the system except system
users. In our deployment we have used the `ForceCommand` option in
`sshd_config(5)` to include the cgroup setup into the login process.
This actually works quite well and the scheduling behaves as expected.
Until `systemd(1)` comes into play.

### Systemd

Systemd has its very own ideas and
concepts for the cgroup hierarchy, and it does not like other players
messing with `/sys/fs/cgroup`. Actually, systemd takes away the whole cgroup feature
from the system administrator completely. One can register a service and have systemd delegate cgroup resource
control for that cgroup subtree only. And that's is.

`systemd-cgls(1)` shows the cgroup structure as set up initially by systemd itself.
`-.slice` is the root slice, which is divided into the `user.slice` and the `system.slice`.
All user processes are attached to cgroups in the `user.slice`, grouped by an individual user slice
for each user `user-<uid>.slice`. Each of them is further split into scopes for each of the user's sessions, and
into a `user@<uid>.service` for session infrastructure.
In the systemd-cgls excerpt below, *usera* with uid 12345 has two sessions: A thinlinc
session and an independent ssh session.

    -.slice
    ├─user.slice
    [...]
    │ ├─user-12345.slice
    │ │ ├─session-8482.scope
    │ │ │ ├─11642 tl-session: usera
    │ │ │ ├─11675 /opt/thinlinc/libexec/tl-xinit /bin/bash -c exec -l "$SHELL" -c...
    │ │ │ ├─11693 /opt/thinlinc/libexec/Xvnc :1 -depth 24 -geometry 3840x1200 -fp...
    │ │ │ ├─11702 /bin/bash /opt/thinlinc/etc/xsession
    │ │ │ ├─12420 /vol/firefox/lib/60.0.1/firefox
    │ │ │ [...]
    │ │ ├─user@12345.service
    │ │ │ ├─at-spi-dbus-bus.service
    │ │ │ │ ├─12100 /usr/lib/at-spi2-core/at-spi-bus-launcher
    │ │ │ │ ├─12105 /usr/bin/dbus-daemon --config-file=/usr/share/defaults/at-spi...
    │ │ │ │ └─12204 /usr/lib/at-spi2-core/at-spi2-registryd --use-gnome-session
    │ │ │ ├─dbus.service
    │ │ │ │ ├─11944 /usr/bin/dbus-daemon --session --address=systemd: --nofork --...
    │ │ │ │ ├─11946 /usr/lib/x86_64-linux-gnu/xfce4/xfconf/xfconfd
    │ │ │ [...]
    │ │ └─session-8484.scope
    │ │   ├─11739 sshd: usera [priv]
    │ │   └─11879 sshd: usera@notty
    │ └─user-12346.slice
    │   ├─user@12346.service
    │   │ ├─gvfs-gphoto2-volume-monitor.service
    │   │ │ ├─2643 /usr/lib/gvfs/gvfs-gphoto2-volume-monitor
    │   │ │ └─2644 /usr/lib/gvfs/gvfs-gphoto2-volume-monitor
    │   │ ├─at-spi-dbus-bus.service
    │   │ │ ├─2629 /usr/lib/at-spi2-core/at-spi-bus-launcher
    [...]

