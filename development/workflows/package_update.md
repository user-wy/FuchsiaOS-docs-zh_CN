<!--# Developing with Fuchsia packages-->
# 使用 Fuchsia 包进行开发

Almost everything that exists on a Fuchsia system is a [Fuchsia package][pkg-struct].
Even the contents of /system are backed by a Fuchsia package. Whether it is
immediately apparent or not almost everything you see on Fuchsia lives in a
package. This document will cover the basics of a package-driven workflow where
you [build][pkg-doc] a package and push it to a Fuchsia device which is reachable
via IP from your development host.

## Pre-requisites and overview

The host and target must be able to communicate over IP. In particular
it must be possible to SSH from the development host to the target device, and
the target device must be able to connect via TCP to the development host on
port 8083. The SSH connection is used to issue commands to the target device.

The development host will run a simple, static file, HTTP server which makes the
updates available to the target. This HTTP server is part of the Fuchsia source
code and built automatically.

The target is instructed to look for changes on the development host via a
couple of commands that are run manually. When the update system on the target
sees these changes it will fetch the new software from the HTTP server running
on the host. The new software will be available until the target is rebooted.

## Building

> TODO(jmatt): improve to talk about wider variety of build options

To build a package containing the required code, a package type build rule is
used. If one of these needs to be created for the target package, consult the
reference [page][pkg-doc] for this. Some build rule types are actually
extensions of the package rule type, for example [`flutter_app`][flutter-gni]
extends the package type. The rule must also not set `deprecated_system_image` to true,
things in the system image can only be updated by [paving][paver].

Once an appropriate build rule is available the target package can be
re-generated by running `fx full-build` or `fx build <target_name>`.

## Connecting host and target

The Fuchsia source contains a simple HTTP server which serves static files. The
build generates a [TUF][TUF-home] file tree which is served.

The update agent on the target does not initially know where to look for
updates. To connect the agent on the target to the HTTP server running on the
development host, it must be told the IP address of the development host.
The host HTTP server is started and the update agent is configured by calling
`fx serve -v` or `fx serve-updates -v`.  `fx serve` will run both the bootserver
and the update server and is often what people use. `fx serve-updates` runs just
the update server. In both cases, `-v` is recommended because the command will
print more output which may assist with debugging. If the host connects
successfully to the target you will see the message `Ready to push packages!` in
the shell on your host.

The update agent on the target will remain configured until it is repaved or
persistent data is lost. The host will attempt to reconfigure the update agent
when the target is rebooted.

## Triggering package updates

In the future certain updates may happen automatically, but today the update
agent on the target must be told to look for an update. To accomplish this a SSH
connection is made from the host to the target and a command is run to tell the
update agent to look for a new package. To trigger the update invoke
`fx build-push <package_name>`. The &lt;package_name&gt; argument can be
repeated to push multiple packages. The &lt;package_name&gt; argument can also
be omitted to cause *all* packages, except the system package, to be pushed.
The number of packages is typically large and the `build-push` mechanism doesn't
scale well. To update all the packages on a target it will be faster to do an
[OTA].

The update package(s) will be available until the target is rebooted. Following
a reboot the package data will still be on local storage, but will be
inaccessible. This is a limitation of the current implementation which will
improve over time. When doing an [OTA] the result of the OTA is persistent
across reboots.

## Triggering an OTA

Sometimes there may be many packages changed or the kernel may change or there
may be changes in the system package. To get kernel changes or changes in the
system package an OTA or [pave][paver] is *required*, `fx build-push` isn't
capable of updating these things. An OTA update will usually be faster than
paving. When updating a large number of packages an OTA update will be faster
than `fx build-push` because of its more optimized implementation.

The command `fx ota` asks the target device to perform an update from any of
the update sources available to it. To OTA update a build made on the dev host to
a  target on the same LAN, first build the system you want. If `fx serve [-v]`
isn't already running, start it so the target can use the development host as an
update source. The `-v` option will show more information about the files the
target is requesting from the host. If the  the `-v` flag was used there should
be a flurry of output as the target retrieves all the new files. Following
completion of the OTA the device will reboot.


## Just the commands

  * `fx serve -v` (to run the update server for both build-push and ota)
    * `fx serve-updates -v` (to run only the update server, not the bootserver)
  * `fx build-push <package_name>` (each time a change is made you want to push)
  * `fx shell "killall sysmgr"` (optional, depending on your component)
  * `fx ota` (to trigger a full system update and reboot)

## Issues and considerations

### You can fill up your disk

Every update pushed is stored in the content-addressed file system, blobfs.
Following a reboot the updated packages may not be available because the index
that locates them in blobfs is only held in RAM. The system currently does not
garbage collect inaccessible or no-longer-used packages (having garbage to
collect is a recent innovation!), but will eventually. Until then, the easiest
solution is to re-pave the device, which will clear out blobfs.

### Restarting without rebooting

If the package being updated hosts a service managed by Fuchsia that service
may need to be restarted. Rebooting is undesirable both because it is slow and
because the package will revert to the version paved on the device. In this
case 'fuchsia' can be restarted. More accurately, `sysmgr` can be restarted.
`sysmgr` can be restarted by running `fx shell "killall sysmgr"`.

### Updating things in the system package

If a package is part of the system image (because its package rule sets
`deprecated_system_image = "true"`) then it can not be updated with the package update flow.
To update the system package, an [OTA] is required which requires a
reboot at present.

The system package is intended for a few key pieces of code and data that are
involved in booting the system. There are very few reasons that code should need
to live in the system package. Being a driver, for example, is not a reason
something should live in the system package. If there is an architectural reason
for something to be in the system package it is likely that either the
architecture is expected to change or a redesign should be considered to remove
the system package constraint.

### Packaging code outside the Fuchsia tree

Packaging and pushing code that lives outside the Fuchsia tree is possible, but
will require more work. The Fuchsia package format is quite simple. It consists
of a metadata file describing the package contents which is described in more
detail in the [Fuchsia package][pkg-struct] documentation. The metadata file is
added to a TUF file tree and each of the contents are named after their Merkle
root hash and put in a directory at the root of the TUF file tree called 'blobs'.

[pkg-struct]: https://fuchsia.googlesource.com/garnet/+/master/go/src/pm/README.md#structure-of-a-fuchsia-package "Package structure"
[TUF-home]: https://theupdateframework.github.io "TUF Homepage"
[pkg-doc]: /development/build/packages.md "Packaging docs"
[flutter-gni]: https://fuchsia.googlesource.com/topaz/+/master/runtime/flutter_runner/flutter_app.gni "Flutter GN build template"
[paver]: paving.md "Fuchsia paver"
[OTA]: #triggering-an-ota "Triggering an OTA"