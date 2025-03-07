Running Package Tests
=====================

Packages will have their own tests under `debian/tests`. We need to run those to ensure there are no regressions.

We can have Launchpad do it against a PPA, or use an LXC container or a VM to run the autopkgtests locally.  Each approach has its benefits, so they're all worth learning, but the first option (PPA-based testing) produces results most similarly to what occurs in the archive itself, so we'll start there.


PPA-Based Autopkgtest Testing
-----------------------------

First, if you haven't already, install `ppa-dev-tools`:

    $ sudo snap install ppa-dev-tools
    $ ppa --help
    usage: ppa [-h] [-C CONFIG_FILENAME] [-D] [-V] [--dry-run] [-v] [-q]
           {create,desc,destroy,list,set,show,status,tests,wait} ...


Next, you'll need to [set up a PPA and build your package in it](PackageBuilding.md) as described in the "Building Binary Packages via PPA" section.  Once it has built binaries for the architecture(s) you intend to test,

  $ ppa tests --show-url ppa:kstenerud/postfix-postconf-segfault-1753470 --release bionic

This prints to the console a bunch of lines like:

  Using Release Packages ♻️ 
  http://autopkgtest.ubuntu.com/request.cgi?release=bionic&arch=amd64&package=postfix&ppa=kstenerud/postfix-postconf-segfault-1753470&trigger=postfix/3.3.0-1ubuntu0.1~ppa1
  http://autopkgtest.ubuntu.com/request.cgi?release=bionic&arch=s390x&package=postfix&ppa=kstenerud/postfix-postconf-segfault-1753470&trigger=postfix/3.3.0-1ubuntu0.1~ppa1
  ...

The autopkgtest requests require special permissions to run; as a new developer you'll need to ask your co-workers or a coredev to load them.  If you don't know where else to ask, the #ubuntu-devel IRC channel is suitable.  The --showurl parameter causes these URLs to be printed out so they're easier to cut-and-paste into email or chat channels.

Once you've gained permissions to run autopkgtests, though, you can load each of these URLs in your web browser yourself, which will cause the appropriate autopkgtests to run.  If you omit the --show-urls parameter, `ppa tests` will instead display clickable links, making it even more convenient.

After a while, run `ppa tests` again to see how the tests are coming along:

  $ ppa tests ppa:kstenerud/postfix-postconf-segfault-1753470 --release bionic
  ...
  Results: (from http://autopkgtest.ubuntu.com/results/.../?format=plain)
    postfix @ amd64:
      14.06.22 21:57:01 ✅     Triggers: postfix/3.3.0-1ubuntu0.1~ppa1
  ...

You can load up the log URLs to see details about why anything failed.


Preparing a Testing Image
-------------------------

You'll need an image to test from. `autopkgtest` will build a suitable image for you. You may want to regenerate the image from time to time to cut down on the number of updates it must run.

The type of image you can use (chroot, container, or VM) depends on the restrictions in `debian/tests/control`(see https://salsa.debian.org/ci-team/autopkgtest/-/blob/master/doc/README.package-tests.rst)

Important restrictions:

 * `breaks-testbed`: This test is liable to break the testbed system. VM or container recommended.
 * `isolation-machine`: You must use a VM to run these tests.
 * `isolation-container`: You must use a VM or container to run these tests.
 * `needs-reboot`: The test reboots the machine, so you must use a VM or container.


#### Building a VM Image

Create an image like this (replacing focal with your release of choice):

    $ autopkgtest-buildvm-ubuntu-cloud -r focal -v --cloud-image-url http://cloud-images.ubuntu.com/daily/server

Note: Use `-m` to specify a closer mirror or `-p` to use a local proxy if it's slow.

Copy the resulting image (autopkgtest-focal-amd64.img) to a common directory like `/var/lib/adt-images`


#### Building a Container Image

    $ autopkgtest-build-lxd images:ubuntu/impish/amd64

You should see an autopkgtest image now when you run `lxc image list`.


### Run the Tests

#### In a VM, Manual

Make sure you're one directory up from your package directory and run:

    $ autopkgtest -U -s -o dep8-mypackage mypackage/ -- qemu /var/lib/adt-images/autopkgtest-focal-amd64.img

Where:

 * `-U`: run apt-get upgrade
 * `-s`: stop and give you a shell if there is a failure. Good to debug
 * `-o dep8-mypackage`: Put your package name in here. Writes output report to the directory dep8-mypackage.
 * `mypackage/`: Put your package name here. The trailing slash tells it to interpret this as a directory rather than a package name.

Everything after the `--` tells it how to run the tests. `qemu` is shorthand for `autopkgtest-virt-qemu`.


#### In a VM, Using the PPA

    $ autopkgtest -U -s -o dep8-mypackage-ppa --setup-commands="sudo add-apt-repository -y -u -s ppa:mylaunchpaduser/focal-mypackage-fixed-something-1234567" -B mypackage -- qemu /var/lib/adt-images/autopkgtest-focal-amd64.img

Where (in setup-commands):

 * `-y`: Assume "yes" for all questions
 * `-u`: Run apt-update
 * `-s`: Add the source line as well
 * `-B`: Don't build

Note: In this case, the package name **doesn't** have a trailing slash because we want to install the package.


#### In a Container, Using the PPA

The command only differs after the `--` part. For example:

    $ autopkgtest -U -s -o dep8-mypackage-ppa --setup-commands="sudo add-apt-repository -y -u -s ppa:mylaunchpaduser/focal-mypackage-fixed-something-1234567" -B mypackage -- lxd autopkgtest/ubuntu/focal/amd64

#### In Canonistack

This is by far the closest in terms of "similarity" to the real autopkgtests since they also run in such an environment.
But it needs some preparation. First of all you must have been *unlocked for* and have set up [Canonistack](https://wiki.canonical.com/InformationInfrastructure/IS/CanoniStack-BOS01) for yourself.

In going through the set up process for Canonistack, you'll have created an openstack RC file that sets region, auth and other environment variables. Go ahead and source this file, if you haven't already.
Then you'd look for the image you want to boot like:

```
$ source ~/.canonistack/novarc_bos01
$ openstack image list | grep -i arm64 | grep hirsute
| 4d24cfbe-b6a5-4d84-8c50-b9f025d0dd43 | ubuntu/ubuntu-hirsute-daily-arm64-server-20201124-disk1.img    | active |
| 1cfeacff-f04a-4bce-ab92-9d8fec7e5edb | ubuntu/ubuntu-hirsute-daily-arm64-server-20201125-disk1.img    | active |
```

Make sure that you have installed `glance`. The `nova` script we will use in the following example needs it. 
You can install it by either using apt: 
```
$ sudo apt update && sudo apt install python3-glanceclient
```
or pip:
```
$ pip install python3-glanceclient
```

[Finally to run the test on Canonistack](https://wiki.ubuntu.com/ProposedMigration#Reproducing_tests_in_the_cloud) is quite similar to the other invocations. Just two things change compared to "local" autopkgtest-runner invocations.

 * `--setup-commands setup-testbed` will have autopkgtest execute `/usr/share/autopkgtest/setup-commands/setup-testbed` on the target which converts any system into a system that is ready for autopkgtest to log in
 * `-- ssh -s nova` achieves two things
  * First of all it selects the ssh virtualization driver `autopkgtest-virt-ssh` to reach out to a remote system
  * Furthermore it selects the setup script `nova` from `/usr/share/autopkgtest/ssh-setup/nova` which happens to know how to deal with openstack

```
    # General pattern
    $ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail <mypackage>.dsc -- ssh -s nova -- --flavor m1.small --image <image> --keyname <yourkeyname>
```

```
    # One example
    $ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail systemd_247.3-1ubuntu2.dsc -- ssh -s nova -- --flavor m1.small --image ubuntu/ubuntu-hirsute-daily-arm64-server-20201125-disk1.img --keyname paelzer_canonistack-bos01
```

You can use usual openstack terms, like other flavors to size the VM that is used or other images to run the same test on different releases or architectures.


### Armhf is special

Canonistack does not have native armhf nodes.
Due to that the autopkgtests on that architecture actually run in armhf containers on arm64 hosts.
To recreate that environment you'd first need to get a canonistack arm64 instance and there combine all of the above like:

```
$ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail <mypackage>.dsc -- lxd ubuntu-daily:mantic/armhf
```

These days normal images mostly work, but for completeness (and because you read this being cursed by tracking a special case) there is also a form which creates an image adapted to the use for autopkgtest.

```
# prep armhf container image for autopkgtest
$ autopkgtest-build-lxd ubuntu-daily:mantic/armhf

# check the created container
$ lxc image list
...
| autopkgtest/ubuntu/mantic/armhf | d5d93f552340 | yes    | autopkgtest Ubuntu mantic armhf       | armv7l       | CONTAINER | 571.57MB | Aug 25, 2023 at 8:56am (UTC) |
...

# run a test in that container
$ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail <mypackage>.dsc -- lxd autopkgtest/ubuntu/mantic/armhf
```

### Common options you'll need

#### Run against -proposed or subsets thereof

Quite often a test fails by running against new packages in the proposed pocket. Then it often will be helpful to check if the test needs other packages from proposed to resolve the issue. That can easily be done via the option `--apt-pocket`.

Commonly a test will run against all packages in release plus the new candidate from proposed, that would look like:

```
    --apt-pocket=proposed=src:yourpkg
```

To run against all packages that are in proposed, you'd simply not refer to a package

```
    --apt-pocket=proposed
```

And if instead you'd need a given set of packages, but not everything else from proposed you can use a comma separated list

```
    --apt-pocket=proposed=src:srcpkg1,srcpkg2
```

Examples testing various combinarions against octave-parallel:

```
# normal
autopkgtest --apt-pocket=proposed --shell-fail octave-parallel_4.0.0-2ubuntu1~ppa1.dsc -- qemu ~/work/autopkgtest-hirsute-amd64.img
# all proposed
autopkgtest --apt-pocket=proposed --shell-fail octave-parallel_4.0.0-2ubuntu1~ppa1.dsc -- qemu ~/work/autopkgtest-hirsute-amd64.img
# specific subset
autopkgtest --apt-pocket=proposed=src:octave,octave-parallel,octave-struct --shell-fail octave-parallel_4.0.0-2ubuntu1~ppa1.dsc -- qemu ~/work/autopkgtest-hirsute-amd64.img
```

#### Size the test VM

Quite often one might wonder "hmm, might this work with more CPU/memory"
At least in the case of qemu and nova that can be controlled.

For `qemu` you can add `--ram-size` and `--cpus`

Example to run the same test in different sizes:

```
autopkgtest --no-built-binaries --apt-upgrade --shell-fail octave-parallel_4.0.0-2ubuntu1~ppa1.dsc -- qemu --ram-size=1536 --cpus 1 ~/work/autopkgtest-hirsute-amd64.img
autopkgtest --no-built-binaries --apt-upgrade --shell-fail octave-parallel_4.0.0-2ubuntu1~ppa1.dsc -- qemu --ram-size=4096 --cpus 4 ~/work/autopkgtest-hirsute-amd64.img
```

For `nova` one has to use [openstack flavors](https://docs.openstack.org/nova/latest/user/flavors.html). If unsure which ones are defined you can check with `openstack flavor list`. An example passing nova different sizes then would be:

```
$ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail systemd_247.3-1ubuntu2.dsc -- ssh -s nova -- --flavor m1.small --image ubuntu/ubuntu-hirsute-daily-arm64-server-20201125-disk1.img --keyname paelzer_canonistack-bos01
$ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail systemd_247.3-1ubuntu2.dsc -- ssh -s nova -- --flavor cpu4-ram8-disk20 --image ubuntu/ubuntu-hirsute-daily-arm64-server-20201125-disk1.img --keyname paelzer_canonistack-bos01
$ autopkgtest --no-built-binaries --apt-upgrade --setup-commands setup-testbed --shell-fail systemd_247.3-1ubuntu2.dsc -- ssh -s nova -- --flavor cpu8-ram16-disk50 --image ubuntu/ubuntu-hirsute-daily-arm64-server-20201125-disk1.img --keyname paelzer_canonistack-bos01
```

#### Restrict networking

Some DEP8 test failures occur due to the autopkgtest environment's
network restrictions.  A good clue of this is when the tests pass
reliably locally yet fail after uploading to Launchpad.  Other clues
include test logs mentioning port errors, inaccessible URLs or IPs, and
upstream language packaging tools like npm, compose, pip, etc.

The network restrictions can be mimicked by invoking autopkgtest locally
with an internal proxy.  This isn't replicating that 100% as there are
also firewalls in place, but if in doubt it often is worth to retry a
local VM based repro with this to check if it fails this way.

What you need to do is adding the internal proxy (needs VPN up); or if you want
to try another proxy of your choice that is rather restrictive. Then add this
to the call of autopkgtest `--env='no_proxy=127.0.0.1,127.0.1.1,localhost,localdomain,novalocal,internal,archive.ubuntu.com,security.ubuntu.com,ddebs.ubuntu.com,changelogs.ubuntu.com,ppa.launchpad.net' --env='http_proxy=http://squid.internal:3128'`

Example when we tracked down an issue with ulfius:
```
$ autopkgtest --env='no_proxy=127.0.0.1,127.0.1.1,localhost,localdomain,novalocal,internal,archive.ubuntu.com,security.ubuntu.com,ddebs.ubuntu.com,changelogs.ubuntu.com,ppa.launchpad.net' --env='http_proxy=http://squid.internal:3128' --no-built-binaries --apt-upgrade --shell-fail ulfius_2.7.1-3.dsc  -- qemu ~/work/autopkgtest-impish-amd64.img
```

### Save the Results

You'll see the tests run:

    autopkgtest [11:47:12]: version 5.3.1
    autopkgtest [11:47:12]: host karl-tp; command line: /usr/bin/autopkgtest -U -s -o dep8-postfix-ppa '--setup-commands=sudo add-apt-repository -y -u -s ppa:kstenerud/postfix-postconf-segfault-1753470' -B postfix -- lxd autopkgtest/ubuntu/focal/amd64
    autopkgtest [11:47:31]: @@@@@@@@@@@@@@@@@@@@ test bed setup

    ...

    ----------------------------------------------------------------------
    Ran 15 tests in 67.027s

    OK
    autopkgtest [11:49:51]: test postfix: -----------------------]
    autopkgtest [11:49:51]: test postfix:  - - - - - - - - - - results - - - - - - - - - -
    postfix              PASS
    autopkgtest [11:49:52]: @@@@@@@@@@@@@@@@@@@@ summary
    postfix              PASS

Save the last part for the description for your merge proposal.
