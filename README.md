of_manifest
===========

# Introduction

This repository provides metadata that allows integration of Connected
Way's products with the Yocto Project build system.

The branches and manifest provided here have been qualified 
against the pyro and hardknott releases of Yocto.

# Supported Distros

There are two supported yocto distributions of poky for Openfiles:
- pyro
- hardknott

Others are likely supported but not currently qualified.

# Background

The intent of of_manifests is to support and demonstrate
Connected Way's Open Files SMB client framework on embedded linux platforms.
Specifcally, we want to demonstrate support on Linux 4.9 and Linux 5.10
kernels.

Linux 4.9 is supported by the poky Pyro distribution and Linux 5.10 is
supported by the poky Hardknott distribution.  Connected Way has forked
the Yocto Project's poky repository branches pyro and hardknott and have
hosted this fork on Connected Way's github.

Our intent was also to demonstrate openfiles support on Arm64 (aarch64)
platforms.  There is no machine specific software within openfiles so the
machine architecture is somewhat arbitrary, but we use Apple Mac Book Pros
based on Apple's M1 chipset (a 64 bit Arm processor).  Because of this, we
want a Ubuntu linux host distribution that supports arm64.  The only two
currently available Ubuntu distributions that support arm64 are
20.04 and 21.04.

Through experimentation, we discovered that the pyro distribution of poky
will not easily run on either Ubuntu 21.04.  It can be made to run on Ubuntu
20.04 but we have had to patch the pyro poky release to accomplish this.
We have integrated the required patches to our own pyro distribution
to support a Ubuntu 20.04 host onto the branch pyro-20.04.  Description of
these changes are discussed in [the meta-connectedway Readme](https://github.com/connectedway/meta-connectedway/blob/main/README.md)

# Deploying Openfiles in Yocto Project's Poky Distribution

## Registering for the Open Files git repos

In order to build the openfiles distribution, you will need to be registered
for the https://github.com/connectedway repos.  If you already have a
public/private keypair you would like to use, you can skip the next step

### Create a public/private key pair

Only do this if you do not already have a public/private key pair that you
would like to use for git.

You can find out more info [here](https://docs.github.com/en/authentication/connecting-to-github-with-ssh)

### Send the public key to Connected Way.

Once we receive your public keys, we will add as deploy keys for Connected
Way's protected git repos.  A public key will typically have a suffix of
.pub.  Protect your private key from unauthorized access.

## Set up your yocto workspace

For background, it is helpful to be familiar with the Yocto Project.  You
can read the [Yocto Quick Start](https://docs.yoctoproject.org/brief-yoctoprojectqs/index.html).

For the openfiles Yocto release, we leverage the repo tool which you can
also find out more about [Here](https://source.android.com/docs/setup/download#repo).

Rather than read, you can follow along here:

Create your workspace:

```
$ mkdir yocto
$ cd yocto
```

Get a copy of the repo tool:

```
$ curl https://storage.googleapis.com/git-repo-downloads/repo > repo
$ chmod a+x repo
```

Initialize your workspace.  We currently support two yocto releases: pyro
(for the 4.9 kernel) and hardknott (for the 5.10 kernel).  We also
support two types of builds: core and smb.  Core is the opensource
openfiles framework.  Smb contains the core support but also adds the smb
client (and server) to the build.  Specify either
pyro or hardknott in the branch argument and the core.xml or smb.xml of
the repo init command below:

For example, to generate a core build of openfiles on pyro, issue:

```
./repo init https://github.com/connectedway/of_manifests -m core.xml -b pyro
./repo sync
```

To generate an smb build of openfiles on hardknott, issue:

```
./repo init https://github.com/connectedway/of_manifests -m smb.xml -b hardknott
./repo sync
```

Repo sync will take a little bit of time as it clones all required repos.

## Configuring a Build Environment for the Yocto Distribution

The correct layers for your selected kernel will automatically be configured
by the repo tool in the previous step.  

Initialize the build environment.

```
$ source oe-init-build-env
```

By default, the openfile distribution from the Connected Way
git repo will build Openfiles support for a qemuarm64
machine.  If you wish to build for a different
target, you will have to modify the `poky/build/conf/local.conf` and
make the following OPTIONAL change:

1. Select the correct target MACHINE.  We use qemuarm64:
```
MACHINE ?= "qemuarm64"
```

# Building A Poky Distribution With OpenFiles

If all the previous steps were successful, building a poky distribution with
openfiles will be one command:

```
bitbake core-image-minimal
```

This will take some time.  It should complete successfully although there
may be some bitbake warnings.  If you are concerned about the warnings,
or if you run into errors during the build, please contact your support at
Connected Way for clarification.  

Once the build is complete, you can run your embedded linux image in qemu
by executing the following command:

```
runqemu.openfiles
```

This script will start the qemu environment with arguments we have determined
provide the best experience.  Once the system boots up, you can log in with
the username root:

```
Poky (Yocto Project Reference Distro) 3.3.6 qemuarm64 /dev/ttyAMA0

qemuarm64 login: root
root@qemuarm64:~#
```

# Explore the File System

Openfiles and openfile artificts will be installed in the following locations:

- /usr/bin/openfiles/smbcp: an openfiles smb aware copy utility (see below).  Only installed on the smb variant of openfiles.
- /usr/bin/openfiles/test_dg: a Datagram loopback test application
- /usr/bin/openfiles/test_event: A openfiles event test
- /usr/bin/openfiles/test_iovec: Tests of vector based messages
- /usr/bin/openfiles/test_path: A path manipulation test
- /usr/bin/openfiles/test_perf: A test of the performance measurement facility
- /usr/bin/openfiles/test_stream: A TCP loopback test application
- /usr/bin/openfiles/test_thread: A test of openfiles threading
- /usr/bin/openfiles/test_timer: A test of openfiles timers
- /usr/bin/openfiles/test_waitq: A test of openfiles wait queues.
- /usr/bin/openfiles/test_fs_linux: A test of local file I/O
- /usr/bin/openfiles/test_fs_smb: A test of remote smb file I/O.  Only installed on an smb variant of openfiles.
- /usr/bin/openfiles/test_all: An aggregate of all tests
- /usr/lib/libof_core_shared.so.1*: The openfiles framework
- /usr/lib/libof_smb_shared.so.1*: The openfiles smb support.  Only installed on an smb variant of openfiles

NOTE: the test_* applications are designed as continuous integration (CI)
applications.  As such they are statically linked and configured statically
during the build.  The static configuration is fine for CI deployment but
not really appropriate in a deployed system.  This becomes an issue with
the test_fs_linux and test_fs_smb applications.  The path to use for the
test is statically configured during the build.  For test_fs_linux it is
configured with `/tmp/openfiles`.  For test_fs_smb it is configured with
`//****:****@192.168.1.206:445/spiritcloud/`.  The static configuration
for test_fs_linux may be fine for running in a non CI environment but
obviously the test_fs_smb configuration will be inappropriate in a non-ci
environment.

Fortunately for both, the default test location can be overridden in the
runtime configuration file which is installed by yocto in `/etc/openfiles.xml`.
If this file hasn't been modified previously, there will be a section in the
file that appears as:

```
  <drives></drives>
```

Simply change this to:

```
  <drives>
    <map>
      <drive>test</drive>
      <description>directory for test_fs_linux or test_fs_smb</description>
      <path>PATH_TO_DIR</path>
    </map>
  </drives>
```

Where PATH_TO_DIR is the path to the temporary directory to use.  Both
test_fs_linux and test_fs_smb will use the same map and the path can specify
either a local or remote file.  A file specification is of the form:

```
[//username:password:domain@server/share/]path.../file
```

Where the presence of a leading `//` signifies a remote location.  The
absence of a leading `//` signifies a local location.  So in the case of
test_fs_linux, we may want the path element to be:

```
      <path>/tmp/openfiles</path>
```

and for test_fs_smb, we may want the path element to be:

```
      <path>//username:password@server/share/openfiles</path>
```

Replace `username`, `password`, `server`, and `share` with values appropriate
for you network topology.

Note that both test_fs_linux and test_fs_smb will create the directory if
it doesn't exist but will only create one level.  In other words, if the
parent directory doesn't exist, the test will fail.

For those curious, test_fs_linux and test_fs_smb are nearly identical
programs.  The main difference between the two is in the way they are
linked.  test_fs_smb will link with the smb support while test_fs_linux will
not.  test_fs_linux can only be run with local paths.  test_fs_smb can be
run with either. Therefore, running test_fs_smb with a local file will
result in the the same test as teset_fs_linux.  Running test_fs_linux with a
remote file will result in a failure.

```
$ ./build-yocto-smbfs/of_core/test/test_fs_linux
1502715539 OpenFiles (main) 5.0 1
1502715539 Loading /etc/openfiles.xml
1502715539 Device Name: localhost
Unity test run 1 of 1
1502715539 Starting File Test with //*****:*****@192.168.1.206/spiritcloud/openfiles

1502716543 Failed to Validate Destination //****:****@192.168.1.206/spiritcloud/openfiles, Not Supported(50)
.

-----------------------
1 Tests 0 Failures 0 Ignored 
OK
Total Allocated Memory 0, Max Allocated Memory 8767
```

# smbcp Example Application

The openfiles manifests will install a sample openfiles smb application
called `smbcp`.  You can find more [Here]((https://github.com/connectedway/smbcp/blob/main/README.md)

