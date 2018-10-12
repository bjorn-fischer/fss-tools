# tl-wrapper [(source)](../tl-wrapper)

tl-wrapper (courtesy of [Cendio](https://www.cendio.com/)) should never be called directly.
Instead, it should be used as a `ForceCommand` by adding the following option to your `sshd_config(5)`:

    # allow for unlocking screensaver on ThinLinc sessions and
    # activating fss by calling confine_user
    ForceCommand /usr/local/bin/tl-wrapper
