# lssp [(source)](../lssp)

This simple shell script checks, if all processes of a user are still attached
to the corresponding cgroup. It can be used to check your own processes, the
processes of another user or all processes (excluding system processes).

    List stray processes, i.e. processes that are not confined in a cgroup.
    Usage: lssp [-v] [-a | <username>]
           -v also report confined processes
           -a list stray processes for all users
           If neither -a nor <username> is given list stray processes
               of current user.
