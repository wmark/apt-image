Minimal Ubuntu images for Docker
================================

Ever wondered why Ubuntu for Docker comes with *systemd* and tools for filesystem management?
Yeah, [me](https://twitter.com/murmosh) too.
These are container images without that fuzz.

![size comparison: Ubuntu for Docker 80 MiB, Ubuntu newgen 60 MiB, yet this one is 40 MiB](https://raw.github.com/Blitznote/apt-image/master/ubuntu-for-Docker-sizes.svg?sanitize=true "container image size comparison chart")

Features
--------

* **small**:
  * 63% the size of ubuntu-debootstrap (:16.04@898cb62b7368)
  * 45% the size of ubuntu (:16.04@44776f55294a)
* comes with *apt-transport-https*
* and latest *curl*
* a bootstrap *[ca-certificates.crt](https://github.com/wmark/docker-curl/blob/master/ca-certificates.crt)*
* latest *signify* for Linux from [Blitznote/signify](https://github.com/Blitznote/signify)
* bzip2, jq, plzip, runit (for its *chpst*), unzip
* with locale ISO.UTF-8 as default

Usage
-----

This is meant as drop-in replacement for ```FROM ubuntu``` and ```FROM ubuntu-debootstrap```.

You can use *curl* right away or start with ```apt-get -q update``` as usual.
HTTPS support is already included in *apt*!

Find examples here:

* [Github](https://github.com/search?q=user%3Awmark+docker-)

### Recommendations

Use lightweight *chpst* (31 kB) instead of *gosu* (2635 kB):

```diff
- gosu myuser syncthing "$@"
+ chpst -u myuser -- syncthing "$@"
- gosu nobody:root bash -c 'whoami && id'
+ chpst -u nobody:root -- bash -c 'whoami && id'
```

To account for differences between *gpg v1* and *gpg v2*
I've created a script for fetching keys from keyservers:

```bash
/usr/bin/get-gpg-key 0xcbcb082a1bb943db 0xa6a19b38d3d831ef \
| apt-key add
```

Regenerate the Images
---------------------

1. Use the packages sources from `/etc/apt/sources.list`.
2. Install the packages listed in `build.manifest` using **apt**.
3. Remove any excess that cannot be used from within an container.

I have published a script which automates that.
[You can find it on Github, as answer to question/issue 2](https://github.com/Blitznote/docker-ubuntu-debootstrap/issues/2#issuecomment-256456602).

### Hints

Leveraging git's "textconv" you can track changes to the archive files.
This merely changes how diffs are displayed.
Run:

```bash
git config --global diff.tar.textconv "tar -tavf"
git config --global diff.tar.cachetextconv true

git config --global diff.debian-tarball.textconv "tar --to-stdout -x ./var/lib/dpkg/available -f"
git config --global diff.debian-tarball.cachetextconv true
```

You can use **screen** and **tmux**, for example for long-running processes on Container Linux distributions.
Run something that does not return first. Then, utilize `script` with `docker exec -ti`:

```bash
# on the host:
docker run -d --name "myenv" blitznote/apt-image:16.04 /bin/bash

# Now use this "permanent environment" like this:
docker exec -ti myenv script -q -c "/bin/bash" /dev/null

# Voila! screen/tmux will work as usual, including redraws on resized terminals.
screen -m -- rtorrent
screen -wipe
screen -r
…
```

Caveats
-------

* Images for architecture *amd64*/x86_64 require instruction set **SSE 4**, which has been introduced in 2007.  
  If you don't have a reasonably recent CPU you will eventually run into the `illegal instruction` error.
  Another symptom of missing instruction sets is message `Sub-process… exited unexpectedly` with its
  accompanying logline `trap invalid opcode` (run `dmesg` or check your syslog daemon for details).
* CPUs preceding *AMD family `15h`* and *Intel's Ivy Bridge* will not work.  
  *Intel Edison* as well as *KNL* is supported.
* You need Linux 4.13.0 or later, with **Seccomp** for sandboxing of processes.  
  `zgrep SECCOMP /proc/config.gz`
