# Multilib (or multiarch?) and Void Linux

## Current state

As it stands, our multilib system lacks flexibility. It only works on x86_64
glibc, where it allows one to use packages from our i686 repository. There is
special treatment inside our package build system for these, ranging from
`--libdir` configure options to the automatic creation of the `*-32bit` variants
when packages for i686 are built (where, instead of putting package contents
into `/usr/lib`, they are moved to `/usr/lib32`).

## Future path

Fortunately, q66 brought up an alternative way of achieving nearly the same
functionality, but in a much more flexible way. It even allows one to mix
different CPU architectures, if they are adventurous enough. The proposal can be
seen [here](https://github.com/void-linux/void-packages/issues/27337). It is, in
ways, similar to how Debian's [multiarch](https://wiki.debian.org/Multiarch)
system works, but with less intrusive changes required in the toolchain and
packages; it even allows us to slowly roll out support, without rebuilding the
universe (though rebuilding the universe is definitely going to happen, due to
[time64](https://musl.libc.org/time64.html) changes - I, for one, am looking
forward to using my 32-bit devices in 2038).

Its main limitations are as follows:

- setup is fully manual (the solution for this is something I intend to discuss
   in this repo).
- one can only mix systems with different word sizes: the "native" system will
   be either 64 or 32-bit, and `/usr/lib64` or `/usr/lib32`, respectively, will
   be a symlink to `/usr/lib`; the additional architecture will take the
   remaining `/usr/lib*` value, for example `/usr/lib32` on a 64-bit system, and
   make it a symlink to `/usr/${some_prefix}/usr/lib`. Furthermore, one can have
   only a single additional architecture; there simply isn't room for more,
   although it should be pretty simple to switch from one prefix to another, if
   desired; just not use more than one at the same time.
- unlike the current multilib, it doesn't allow installation of package data or
   binaries into `/usr/share` or `/usr/bin`, since the only place where the
   alternative architecture touches the main system is `/usr/lib${word_size}`.

Things that still have to be worked out:

- are there any libraries (or executables which are only available for a single
   arch) that use helpers from `/usr/libexec`, and which need to be redirected
   to `/usr/${some_prefix}/usr/libexec`?
- some currently mulitlib-aware launchers (such as `/usr/bin/wine` from the
   `wine-common` package) will have to be fixed for the new prefixes and/or
   paths.
- packages like `nvidia`, which manually construct `-32bit` sub-packages and
   depend on the current multilib will have to be fixed.

More things that have to be worked out, but specifically regarding user
experience:

- how does one first create the new architecture prefix?
   - if we make a helper for this, how should it determine the repository to
      use? After all, we have `current/`, `current/musl/` and `aarch64/` as
      possibilities (and complicating matters further, some mirrors might not
      include all of those).
   - which packages should be installed by default in this prefix? Maybe we
      should create a `base-multilib` meta-package?
   - how (if ever) do we transition current glibc multilib users to the new
      multilib system? Can this be automated?
   - given possible conflicts with the cross toolchains, which prefix should we
      use? `/usr/multiarch/${arch_name}` has been suggested and looks promising.
   - how can binaries and package data be made available to the native system?
      - binaries:
         - the same helper creates symlinks in `/usr/bin` for binaries from
            `/usr/${some_prefix}/usr/bin`. Has the disadvantage of being
            complicated to track and "polluting" the namespace.
         - the same helper adds a file in `/etc/profile.d` that adds
            `/usr/${some_prefix}/usr/bin` to `PATH` so it overrides any native
            binaries. Has the disadvantage of being very unpredictable.
         - the same helper creates (or installs a package that contains) a
            "launcher" to fix up `PATH` and then re-launch `SHELL`.
      - `.desktop` files:
         - no idea?
      - man pages:
         - the same helper can edit `/etc/man.conf` to add
            `/usr/${some_prefix}/usr/share/man`.
      - helper binaries:
         - `/usr/libexec` can become `/usr/libexec${word_size}` or instead be
            transferred into `/usr/lib${word_size}/libexec`?
- how does updating and installing new packages in this prefix look? Should the
   same helper fullfil this function? Could updates be hooked into a single
   command so normal system maintenance can be kept simple while updating the
   native system + all prefixes?
- would it be simple to have a tool to switch what `/usr/lib${word_size}` points
   to, so one can switch between multiple architectures easily?
   - one possible solution here is putting together the "launcher" idea and have
      it also enter a user namespace and bind mount `/usr/lib${word_size}` to
      the appropriate `/usr/${some_prefix}/usr/lib`.

Further notes:

- a helper program will also have to set up the dynamic linker symlink in
   `/usr/lib`, which should be pretty simple, and, at least for musl, the
   dynamic linker configuration file under `/etc/ld-musl-$ARCH.path`.
