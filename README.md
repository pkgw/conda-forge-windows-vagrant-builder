# Vagrant-based conda-forge Windows Builder

This repo provides a system that lets you use a Linux machine to build
[conda-forge](https://conda-forge.org/) software packages for the
[Windows](https://www.microsoft.com/en-us/windows) operating system. It does
this by setting up a [Vagrant](https://www.vagrantup.com/) “box” (virtual
machine) that does the actual package building.


## Preparation

In order to use this system, you of course need to have Vagrant installed. The
Ruby installation used by your Vagrant install needs to have the
[winrm-elevated](https://rubygems.org/gems/winrm-elevated/versions/1.1.0) Gem
installed so that it can communicate with Windows guests using the
[WinRM](https://docs.microsoft.com/en-us/windows/desktop/winrm/portal)
protocol. This Gem is not provided by at least some Linux distrbutions (e.g.,
Fedora 28). On Fedora, the following command will install it:

```
sudo gem install --no-user-install --minimal-deps -r -i /usr/share/gems winrm-elevated
```

Note that this command will install non-package-managed files into system
directories.

You also need to specify or create the directory that will contain the files
for the various feedstocks that you intend to build within the Windows Vagrant
box. This directory should be an item named `feedstocks` created in the
directory containing this file, but it can be a symbolic link to some other
directory, e.g.:

```
ln -s ~/src/conda feedstocks
```

The Vagrant box will only be able to access directories below this prefix, so
choose something that will contain all of the feedstock directories you care
about.

You will also need about 11 GiB of free disk space to store the custom Windows
VM image and create the specific conda-forge builder box.


## Usage

All tools provided by this repository are accessed through the script
`./driver.sh`, which provides various “subcommands” like `git`.

The first subcommand you must run is `setup`, which prepares to create the
Vagrant box:

```
./driver.sh setup
```

Once that is done, the most important subcommand is `build`, which uses
`conda-build` to try building a package in the Windows box. You pass it the
path to a feedstock directory, *which must be relative to the “feedstocks”
directory* set up above:

```
./driver.sh build feedstocks/fontconfig-feedstock
```

The first time you run this command (or any others that interact with the
Vagrant box), Vagrant will set up the builder box, which involves a large
download and takes a long time. Once the box has been set up, however,
starting it back up is relatively quick.

Also note that you must have
[rerendered](https://github.com/conda-forge/staged-recipes/wiki/conda-smithy-rerender)
the feedstock so that Conda-build actually thinks that there is anything to do
for Windows. Specifically, there should be files within the feedstock whose
names match the shell glob `.ci_support/win_*.yaml`. You can do this rerender
on your Linux box.

There are also other helper commands. One will perform a package search within
the Windows box, which is helpful for investigating support libraries provided
by MSYS2. For example:

```
./driver.sh search gettext
```

Another one will print out the URLs to the tarballs associated with a particular
package, which is helpful for investigating exactly which files it provides:

```
./driver.sh urls m2w64-gettext
```

The `purge` command will run `conda build purge`:

```
./driver.sh purge
```


# Implementation Notes

This was surprisingly painful to get working. In this section are some notes on the
experience.

### Communicating with the box

This Vagrant setup is aimed at headless, fully automated operation. This means that
there is basically only one way to interact with the Windows VM: via `vagrant ssh`.
Ruled-out options are:

- `vagrant rdp` works, but uses interactive graphics
- `vagrant powershell` refuses to work unless the *host* OS is Windows as well.

Unfortunately, in the VM images I’ve tried this far, the interactive SSH shell
is *incredibly* limited. First, it doesn’t provide an actual (pseudo)TTY
interface (i.e., `tty` prints `not a tty`). This means that line editing
barely works and programs that want interactivity fail. Second, virtually no
shell commands are provided. Commands that are **missing** include `cp`,
`more`, and `cat`. When desperate, you can work around this by using
Powershell. The following kinds of constructs usually work:

```
$ powershell -Command 'cp file c:\vagrant\'
```

So Powershell does work, and you can invoke it remotely through `vagrant ssh`,
but it only runs *non-interactively*.

### Provisioning the box

As far as I have discovered, there are two ways to provision the Windows box.
By default, Vagrant will communicate with the box using SSH, and your
provisioning scripts are Bourne shell scripts — that use the incredibly
limited shell described above. Alternately, you can set Vagrant to communicate
with the box using WinRM, in which case your provisioning script is
PowerShell.

In principle, the SSH approach can work just as well since you can execute
PowerShell scripts from the extremely limited SSH:

```
powershell -ExecutionPolicy Bypass -File 'C:\Vagrant\provision.ps1'
```

However, for whatever reason, when using the `ssh` method in the provisioning
stage, all of the output from the provisioning commands just disappears. This
makes debugging tedious, to say the least. Therefore I've been using the WinRM
approach, with the following bits in the Vagrantfile:

```
config.vm.communicator = :winrm
config.vm.provision :shell, path: "provision.ps1"
```

### Choice of base image

There are a variety of Windows images on the Vagrant Cloud. This includes one
officially provided by Microsoft:
[EdgeOnWindows10](https://app.vagrantup.com/Microsoft/boxes/EdgeOnWindows10).
Unfortunately this image has several important issues that make it challenging
to work with:

- As of writing (August 2018), it’s more than two years old
- It has the extremely limited SSH server/environment described above
- It doesn't expose WinRM on startup, so all provisioning must go through
  the limited SSH environment that produces no output

The
[senglin/win-10-enterprise-vs2015community](https://app.vagrantup.com/senglin/boxes/win-10-enterprise-vs2015community)
box is even older, but it exposes WinRM and has VS2015 preinstalled, which
makes life a lot easier. So that’s what I’ve been using for now.