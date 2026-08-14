# Building a deb package

Here is a summary of the Debian/Ubuntu → sbuild → Launchpad PPA workflow, adapted from a setup implemented for rakarrack-plus.

Ubuntu packaging workflow with a Launchpad PPA

1. Create and register the GPG key

The GPG key is used to sign source packages uploaded to Launchpad.

Creation
```
$ gpg --full-generate-key
```

Then verify:
```
$ gpg --list-secret-keys --keyid-format LONG
$ gpg --fingerprint "Yann Collette"
```

The full fingerprint consists of 40 hexadecimal characters, usually displayed in 10 groups of 4:
```
1234 5678 9ABC DEF0 1234 5678 9ABC DEF0 1234 5678
```

Publish the public key
```
$ gpg --keyserver hkps://keyserver.ubuntu.com \
--send-keys YOUR_FINGERPRINT
```

Only the public key is sent to the keyserver.

The private key remains in:
```
~/.gnupg/
```

Add the key to Launchpad

In Launchpad, add the fingerprint to your account's OpenPGP keys.

Launchpad may send an encrypted confirmation email. If necessary:
```
$ gpg --decrypt message.asc
```

There may be a propagation delay between keyserver.ubuntu.com and Launchpad.

Create a backup

Once the key is working:
```
$ gpg --armor --export-secret-keys "Yann Collette" \
> ~/yann-collette-launchpad-secret.asc
```

and:
```
gpg --armor --export "Yann Collette" \
> ~/yann-collette-launchpad-public.asc
```

The secret file must be stored securely. 2. Create the PPA

In Launchpad, create, for example:
```
ppa:ycollet/audinux
```

The PPA can then contain multiple packages:
```
ppa:ycollet/audinux
├── rakarrack-plus
├── ntk
├── ...
```

For the end user:
```
$ sudo add-apt-repository ppa:ycollet/audinux
$ sudo apt update
$ sudo apt install rakarrack-plus
```

The PPA can be configured for multiple Ubuntu series, for example Noble and Resolute.

3. Prepare the source package

The project must have a directory:
```
debian/
```

containing, in particular:
```
debian/
├── changelog
├── control
├── copyright
├── rules
├── source/
│   └── format
└── patches/
```

For CMake, a minimal debian/rules usually suffices:
```
#!/usr/bin/make -f

%:
dh $@ --buildsystem=cmake
```

Don't forget:
```
$ chmod +x debian/rules
```

The debian/control file must include the necessary Build-Depends.

4. Manage patches

For a package using:
```
3.0 (quilt)
```

patches go in:
```
debian/patches/
```

with:
```
debian/patches/series
```

Example:
```
0001-fix-cmake-build.patch
0002-fix-install-path.patch
```

Create a patch:
```
$ quilt new 0001-fix-cmake-build.patch
# Make changes to CMakeLists.txt
$ quilt add CMakeLists.txt
```

Modify the file, then:
```
$ quilt refresh
```

To add information:
```
$ quilt header -e
```

Typically:
```
Description: Fix build on Ubuntu
Author: Yann Collette <...>
Forwarded: no
```

Patches are automatically applied by dpkg-buildpackage. 5. Build the source package

In the `rakarrack` directory, run the command:
```
$ git archive --format=tar.xz --prefix=rakarrack-1.4.1 v1.4.1 -o ../rakarrack_1.4.1.orig.tar.xz

```

The changelog must contain a Debian version suitable for the PPA:
```
rakarrack-plus (1.4.1-1~ppa1~resolute) resolute; urgency=medium

* Initial PPA build. 

-- Yann Collette <...>  Thu, 13 Aug 2026 13:00:00 +0200
```

Then:
```
$ debuild -S -sa
```

This produces, among other things:
```
../rakarrack-plus_1.4.1.orig.tar.xz
../rakarrack-plus_1.4.1-1~ppa1~resolute.debian.tar.xz
../rakarrack-plus_1.4.1-1~ppa1~resolute.dsc
../rakarrack-plus_1.4.1-1~ppa1~resolute_source.changes
```

`debuild` signs the files with your GPG key.
```
-sa
```

For the first upload of an upstream version, use:
```
$ debuild -S -sa
```

`-sa` forces the inclusion of the upstream `.orig.tar.*` tarball.

For a new Debian upload of the same upstream version, `debuild -S` is usually sufficient if Launchpad already has the tarball.

6. Verify the package locally

For a quick build directly on your machine:
```
$ debuild -b -us -uc
```

This produces a `.deb` file in the parent directory. Installation:
```
$ sudo apt install ../rakarrack-plus_*.deb
```

Simulation:
```
$ apt install --simulate ../rakarrack-plus_*.deb
```

7. Using ccache

For repeated local builds, ccache can significantly speed up recompilations.

Installation:
```
$ sudo apt install ccache
```

With CMake:
```
$ cmake -B build \
-DCMAKE_C_COMPILER_LAUNCHER=ccache \
-DCMAKE_CXX_COMPILER_LAUNCHER=ccache
```

Then:
```
$ cmake --build build
```

Statistics:
```
$ ccache -s
```

I would reserve ccache for local development builds rather than the build intended for Launchpad.

8. Performing a chroot build with sbuild

This is a crucial step to verify that the package actually works in a clean Ubuntu environment.

After creating the source package:
```
$ debuild -S -us -uc
```

Preparing the chroot:
```
$ sudo sbuild-createchroot \
--arch=amd64 \
--components=main,universe,multiverse,restricted \
noble \
/srv/chroot/noble-amd64 \
http://archive.ubuntu.com/ubuntu
```

```
$ sudo sbuild-createchroot \
--arch=amd64 \
--components=main,universe,multiverse,restricted \
resolute \
/srv/chroot/resolute-amd64 \
http://archive.ubuntu.com/ubuntu
```

Then:
```
$ sbuild --dist=resolute ../rakarrack-plus_*.dsc
```

If necessary, add the PPA:
```
sbuild -d resolute \
--extra-repository="deb https://ppa.launchpadcontent.net/audinux/audinux/ubuntu resolute main" \
rakarrack-plus_1.4.1-1~ppa1~resolute.dsc
```

sbuild:
```
creates/uses the Resolute chroot;
reads Build-Depends;
installs dependencies;
compiles;
builds the .deb;
allows detection of missing dependencies.
```

This is much more representative of the Launchpad build than a build performed directly on your machine.

You can also have multiple environments:
```
noble-amd64
resolute-amd64
```

and test:
```
$ sbuild --dist=noble ...
$ sbuild --dist=resolute ...
```

9. Avoid re-downloading dependencies

Install:
```
$ sudo apt install apt-cacher-ng
```

Then configure APT to use:
```
http://127.0.0.1:3142
```

with:
```
Acquire::http::Proxy "http://127.0.0.1:3142";
```

The idea is to have:
```
                Internet
                    │
                    ▼
             apt-cacher-ng
                    │
          ┌─────────┼─────────┐
          ▼         ▼         ▼
       sbuild     sbuild     apt
       Noble     Resolute    host
```

The already downloaded .deb files are then reused.

10. Upload the package to Launchpad

You can check the current configuration:
```
$ cat ~/.dput.cf
```

If it doesn't exist, you can add a Launchpad entry.

For example:
```
[ppa]
fqdn = ppa.launchpad.net
method = ftp
incoming = ~ycollet/audinux/ubuntu/
login = anonymous
allow_unsigned_uploads = 0
```

Once the source package has been validated by sbuild:
```
$ dput ppa:ycollet/audinux \
../rakarrack-plus_1.4.1-1~ppa1~resolute_source.changes
```

Launchpad verifies the signature and then builds the package.

You can track the build from the PPA page.

Do not upload the .deb file: for a PPA, you upload the signed source package (.changes, .dsc, tarballs, etc.).

Launchpad then builds the binaries for the configured architectures.

11. Install from the PPA

Once the build is complete and published:
```
$ sudo add-apt-repository ppa:ycollet/audinux
$ sudo apt update
$ sudo apt install rakarrack-plus
```

Then, normal updates are handled via APT:
```
$ sudo apt update
$ sudo apt upgrade
```

12. Update the package

New packaging revision

For example, to fix an issue without upstream changes:
```
$ dch -i
```

which might produce:
```
rakarrack-plus (1.4.1-1~ppa2~resolute) resolute; urgency=medium
```

Then:
```
$ debuild -S
$ sbuild --dist=resolute ../rakarrack-plus_*.dsc
$ dput ppa:ycollet/audinux ../rakarrack-plus_1.4.1-1~ppa1~resolute_source.changes
```

New upstream version

For example, 1.4.1 → 1.4.2:
```
$ uscan
```

or:
```
$ uscan --download-current-version
```

Then update debian/changelog:
```
rakarrack-plus (1.4.2-1~ppa1~resolute) resolute; urgency=medium
```

and run again:
```
$ debuild -S -sa
$ sbuild --dist=resolute ../rakarrack-plus_*.dsc
$ dput ppa:ycollet/audinux ../rakarrack-plus_1.4.1-1~ppa1~resolute_source.changes
```

13. Complete workflow

Ultimately, your workflow can be summarized as follows:
```
                    UPSTREAM
                       │
                       ▼
                    uscan
                       │
                       ▼
              ┌─────────────────┐
              │ debian/         │
              │ control         │
              │ rules           │
              │ changelog       │
              │ patches/        │
              └────────┬────────┘
                       │
                       ▼
                debuild -S -sa
                       │
                       ▼
             source package signé
                       │
             ┌─────────┴─────────┐
             │                   │
             ▼                   ▼
       build local           sbuild
       + ccache           chroot propre
             │                   │
             └─────────┬─────────┘
                       │
                       ▼
                  BUILD OK
                       │
                       ▼
             dput ppa:ycollet/audinux
                       │
                       ▼
                    LAUNCHPAD
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
            Noble   Resolute   autres
              │        │
              └────────┴────────┘
                       │
                       ▼
                    .deb
                       │
                       ▼
             apt install / upgrade
```

The three commands to remember:

For a pre-prepared package:
```
$ debuild -S -sa
```

→ creates the signed source package
```
$ sbuild --dist=resolute ../rakarrack-plus_*.dsc
```

→ verifies the build in a clean environment
```
$ dput ppa:ycollet/audinux ../rakarrack-plus_1.4.1-1~ppa1~resolute_source.changes
```

→ uploads to Launchpad

This is essentially the workflow you are setting up for rakarrack-plus, and it is perfectly suited for maintaining multiple audio packages in your PPA.

14. Interacting with Launchpad via Python

```
from launchpadlib.launchpad import Launchpad

lp = Launchpad.login_anonymously(
'audinux-build-status',
'production',
version='devel'
)

archive = lp.distributions['ubuntu'].getArchive(
owner=lp.people['ycollet'],
name='audinux'
)

for b in archive.getBuildRecords(
source_package_name='ntk',
distro_arch_series=lp.distroseries['ubuntu'].getSeries(name='resolute')):
print(b.buildstate, b.build_log_url)
```