# confine_user [source](../confine_user)

This shell script creates a cgroup for a user. It is designed to be used
with *sudo(1m)* and should be executed during the login process. **Caveat emptor:
The software was written with security in mind but has never been audited thoroughly.**

Root can use the script to reconfine users who have processes that have
dropped out of the user's cgroup somehow. Just run `confine_user` with the
login name as the first argument.

You may use this snippet for your sudo configuration:

    #
    # /etc/sudoers.d/confine_user
    #
    # Enable all users to call confine_user as root without password
    #

    %users		ALL = NOPASSWD: /usr/local/bin/confine_user
