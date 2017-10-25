# Automatic ZFS Home Directories on FreeBSD

## Introduction

This guide will take you through the steps required to set up FreeBSD as a file
server joined to Active Directory.  For the Active Directory part, you should
follow this [guide to set up Active Directory on FreeBSD](http://jornane.no/doc/ad-auth/).

This guide will work on any stock FreeBSD installation with ZFS.  When
installing FreeBSD, it will ask you if you want UFS or ZFS - choose ZFS and it
will set up booting from ZFS for you.  You can boot from your data disks this
way, even if you use ZFS to do RAID10, RAIDZ or even RAIDZ3.


## Motivation

The challenge with home directories in an Active Directory setup is that users
have two ways of populating their home directory, and they expect both to work
just fine.  A user may choose to log in via Samba, which gives Samba the
oppurtunity to create the home directory.  A user may also choose to log in via
SSH, which keeps Samba at bay but allows PAM to create the home directory.  The
goal of this project is to provide a consistent behaviour regardless which
method the user chooses first.


## Shell script

A lot of the magic in automatically provisioning home directores for users we
will do in a shell script.  This script will check if the home directory exists,
if it doesn't it will check if the home directory is set to be placed directly
under the ZFS volume for home directories (according to
[getent(1)](https://www.freebsd.org/cgi/man.cgi?query=getent(1))) and if
everything is in order it will create the ZFS volume and copy the skel directory
over.  The user will have `diff`,`hold`,`release`,`send`,`snapdir` permissions
awarded after creation of the directory.

The shell script must be copied to `/usr/local/libexec/zfs-mkhomedir`, so that
it is available for both Samba and PAM to be run.

	# fetch -o /usr/local/libexec/zfs-mkhomedir \
		https://github.com/fyrkat/zfsroot/raw/master/zfs-mkhomedir
	# chmod +x /usr/local/libexec/zfs-mkhomedir


## ZFS

For this guide, we will assume that your pool is named `zroot` (FreeBSD
installer default) and that you will have datasets for home directory under
`zroot/home/FYRKAT`, where `FYRKAT` is your NetBIOS name.  We will assume that
you will put the home directories under `/usr/home/FYRKAT`, which is also
available under `/home/FYRKAT` due to default symlinks in FreeBSD.

Create the dataset, make sure you're root for this one.  These commands will
work fine on a fresh FreeBSD, but take care if you've made your own
modifications regarding home directories.

### Preparation

	# mv /usr/home /usr/home.old
	# zfs create -o mountpoint=/usr/home zroot/home
	# mv /usr/home.old/* /usr/home

### To optimize for FreeBSD/Linux use

	# zfs create \
		-o canmount=off \
		-o aclmode=passthrough \
		-o aclinherit=passthrough \
		-o normalization=formD \
		-o casesensitivity=sensitive \
		zroot/home/FYRKAT

### To optimize for Windows use

	# zfs create \
		-o canmount=off \
		-o aclmode=passthrough \
		-o aclinherit=passthrough \
		-o normalization=formKC \
		-o casesensitivity=insensitive \
		zroot/home/FYRKAT

### Background information and explanation

The moving is to preserve any existing home directories, for example the one you
made during the installation.  The ACL settings are so that it is possible to
set ACLs, which Samba needs to apply Windows ACLS.  The normalization is so that
Unicode in filenames is normalized, so you can't end up in a situation where you
have two files with seemingly the same filename but one of them you can't get
rid of.  `formD` means that filenames that visually look identical are identical.
Another option would be `formKC`, which would make filenames that are logically
identical are identical.  For example, under `formKC`, `Office` and `Oﬃce` are
identical, while under `formD` they are not.

Windows is case insensitive, while most \*NIX systems are case sensitive.
Choose the case sensitivity settings that you think most users will benefit
from, although you can favour FreeBSD/Linux if you are unsure, because Samba
will take care of Windows' case insensitivity anyway.

Generally, it makes sense to combine `formKC` with `insensitive`, or `formD`
with `sensitive`.

The volume containing the home directories for AD users doesn't need to be
mounted, hence the `-o canmount=off`.  It is not inheritable, so the actual home
directories *will* be mountable.


## Samba configuration

You already have created a `/usr/local/etc/smb4.conf` file from following the
Active Directory guide, but now we need some modifications.  Set the `[homes]`
section to the following:

	[homes]
		comment = Home Directories
		browseable = no
		writable = yes

		valid users = @"domain users"
		invalid users = root administrator
		root preexec = /bin/sh -c "/usr/local/libexec/zfs-mkhomedir zroot/home/FYRKAT %U"

		# Needed for Folder Redirection GPO
		# from https://mywushublog.com/2012/05/zfs-and-acls-with-samba/
		inherit acls = Yes
		inherit owner = Yes
		vfs objects = zfsacl

**NOTE**: Make sure that you change `zroot/home/FYRKAT` to match your own setup.

The `root preexec` part will make sure that the home directory is created if it
doesn't exist yet.  The `inherit` settings will make Samba behave more like
Windows expects it to do, and the `vfs objects` setting will make Samba work
together with ZFS for storing advanced security attributes for files.  You can
set advanced security attributes from Windows when you open the Properties
dialog for a file or directory.

Remember to restart Samba when you're done.

	# service samba_server restart


## PAM configuration

PAM configuration is actually quite easy, you just need to add one line to the
appropriate PAM file(s), as the last line in the **sessions** section.

	session         required        pam_exec.so /bin/sh -c "/usr/local/libexec/zfs-mkhomedir zroot/home/FYRKAT $PAM_USER"

You can find your PAM files in `/etc/pam.d` for system services or
`/usr/local/etc/pam.d` for services installed through pkg or ports.
Which files are appropriate depends on your setup, but typically you will want
to add it to these when you think a user may use that method to log in:

* **/etc/pam.d/sshd** (upon SSH login)
* **/etc/pam.d/login** (upon local terminal login)
* **/etc/pam.d/xdm** (upon local graphical login)
* **/usr/local/etc/pam.d/kdm** (upon local graphical login)
* **/usr/local/etc/pam.d/gdm** (upon local graphical login)

Of course, if you don't expect your user to log in using one of these methods,
you shouldn't add the user there.

You **must** remove any `pam_mkhomedir.so` lines that you may have added
earlier.  Keeping it in place will prevent `zfs-mkhomedir` from doing it's job.


## Troubleshooting

If you've followed this guide, but home directories are not created, make sure
that you've matched the argument to `zfs-mkhomedir` in PAM and Samba with your
actual ZFS setup.  Furthermore, make sure that the `template homedir` in your
[smb4.conf](https://www.freebsd.org/cgi/man.cgi?query=smb4.conf(5)) matches the
`mountpoint` attribute in ZFS.  Use
[getent(1)](https://www.freebsd.org/cgi/man.cgi?query=getent(1)) to find out
what home directory Samba sets.

I've written this guide to the best of my knowledge, but you may run into
problems that I haven't forseen.  Please let me know when this happens, even if
you managed to solve the problem yourself.  If I know about it, I can add
warnings to this guide so that nobody has to repeat the same error.
