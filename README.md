# fss-tools
Tools and scripts for deploying the Linux fair share scheduler on
multi-user systems, scheduling users against each other instead of processes.
This covers the deployment on [ThinLinc](https://www.cendio.com/thinlinc/what-is-thinlinc "ThinLinc by Cendio") agent hosts. Other scenarios may apply.

The shell scripts [`confine_user`](doc/confine_user.md) and [`tl-wrapper`](doc/tl-wrapper.md) are used to set up the user cgroup during the login process.

[`lssp`](doc/lssp.md) is a small shell script to check whether all processes are confined properly.

And finally, [`watlahm`](doc/watlahm.md) is a GUI tool written in Python 3 and GTK. This tool resembles a top(1) like application which is cgroup aware and is primarily for the users on the system.

If you deploy fair share scheduling using these scripts on a system with `systemd(1)`, you will probably run into some issues,
most notably `systemd` and `onfine_user` fighting over control of the users processes. See below for possible mitigations.

## Motivation

On our site we are currently experimenting with fair share scheduling
on ThinLinc agent hosts. At the Center for Biotechnology (CeBiTec) at Bielefeld University
we provide scientist's desktops (internet and office applications,
and scientific applications) for researchers and teachers.

### Use Case

Formerly based on Sun Ray technology (using Solaris' FSS), we now use a ThinLinc-4.9 setup
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

### Solution: Fair Share Scheduling

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
messing with `/sys/fs/cgroup`. Actually, systemd takes away the cgroup feature
from the system administrator completely. One can register a service and have systemd delegate cgroup resource
control for that cgroup subtree only. And that's it.

#### Divide et impera

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

#### Good leaders delegate

Systemd creates a cgroup for each unit, i.e. every slice, scope and service in the tree above, and puts all
processes belonging to that unit into the corresponding cgroup. Processes cannot be assigned to different cgroups with the same controller. So if you have created your own cgroup `cpu:/user/usera` and attached all of usera's processes to that cgroup, systemd will grab them and put them back into its own cgroup hierarchy -- unless you use the [delegation](https://github.com/systemd/systemd/blob/master/docs/CGROUP_DELEGATION.md) feature of systemd units. By properly setting the `Delegate` property you can exclude a unit (and all its sub-units) from systemd's cgroup resource control.

Unfortunately, the delegate feature [is not available for slices](https://github.com/systemd/systemd/blob/master/docs/CGROUP_DELEGATION.md#some-donts), by design. Otherwise it would be quite simple to delegate the cgroup resource control for the whole `user.slice` and deploy fair share scheduling as mentioned above. But it even gets worse. It seems that scopes are not templatable like the `user@.service`. In the tree above you can template the `user@12345.service` (*any* `user@.service`, i.e.) by creating `/etc/systemd/system/user@.service`. In that file you can use the `Delegate` property and systemd would not touch the processes in that unit any more. But we need *all* the user's processes under our control.

Another way of setting properties of systemd units is `systemctl set-property`, but in this case it will not help at all as just the Delegate property is not settable:

    root# systemctl show user@12345.service
    [...]
    IPEgressPackets=18446744073709551615
    Delegate=yes
    CPUAccounting=no
    [...]
    root# systemctl set-property user@12345.service Delegate=pids
    Failed to set unit properties on user@12345.service:
    Cannot set property DelegateControllers, or unknown property.

#### Using systemd resource control directly

The next thing I tried was using systemd's native [Control Group Interfaces](https://www.freedesktop.org/wiki/Software/systemd/ControlGroupInterface/). You can set properties like `CPUAccounting` and `CPUShares` directly for service units.
This actually works fine for  service units like http servers, but it seems unfit for deploying fair share scheduling.

#### Deviant Methods

My last approach was to disable userland systemd completely by removing `pam_systemd.so` from the pam configuration.
