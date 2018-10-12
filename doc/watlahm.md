# watlahm [(source)](../watlahm)

On a multi-user system with fair share scheduling enforced,
a user should not be able to slow down the whole system. But the user
still may induce lags within his/her own session.
When a user experiences lags or sluggishness `watlahm` may help to
identify the problem.

`watlahm` is written in **Python 3** and requires **Gtk-3.0, GObject** and **Pango**.

As a `top(1)`-like application it provides a tabular overview of either all users
on the system or one's own processes.

## User View

The user view has four data columns: *CPU*, *CPU Average*, *Memory* and *Cache*. *CPU* shows
the current percentage of time slices the user consumes in relation to the maximal number
of time slices available on the whole system (all cores, all HT-units). *CPU Average* is the same
sampled over the last 10 seconds. *Memory* is the percentage of occupied RAM in relation to the memory limit
set up in confine_user. In contrast to the CPU columns, 100% does not mean all RAM of the system, but all RAM
of the user's memory quota. *Cache* also adds to the user's memory quota. It is shown separately since
*Cache* is not only buffer cache, but also includes files on tmpfs.

When a user hits his memory quota (*Memory* + *Cache* ~ 100%) the system will start swapping. As general I/O is not yet
controlled by the cgroup configuration, users hitting a too small memory quota may affect other users as well.

The basic idea behind the user overview is to have some sort of social quota. "How is my resource consumption compared
to other users?"

## Process View

This view is directly comparable to the output of `top(1)` showing your own processes, only. This should help
a user to identify rogue processes.
