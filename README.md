# Bastille
Bastille is a small utility designed to simplify creating and updating jails on 
modern versions of the FreeBSD operating system. 

Bastille lets jails share a common base of binaries, libraries and other files.
Individual jails do not need to modify these resources, hence it suffices
to install them only once. Bastille greatly simplifies sharing resources by
creating base file systems, which contain the necessary system components for 
running jails.  The base file systems consists of a common read-only part and
a writable part of which every jail will receive a unique copy.  Additionally,
Bastille supports templates. A template's content can be merged into a new jail
thus overriding or adding configuration files.

Bastille was inspired by [ezjail](http://erdgeist.org/arts/software/ezjail/),
but the scope of Bastille is much more restricted. Bastille does not provide its 
own service, relying instead on FreeBSD's builtin jail service to start and stop 
jails at boot time. Bastille only works with official binary of FreeBSD. It does 
not manage encrypted jails, or ZFS-datasets.  By default Bastille stores jail 
definitions in jail.conf(8) (a file other than /etc/jail.conf may be used), 
which is now the recommended way to define jails. 

# Base filesystems, templates and jails
The concept of base file systems, templates and jails are central to 
understanding Bastille. 

The base file systems contain FreeBSD's system components. Templates consist of
files that supersede or complement files from the base file systems. Jails 
combine the base file systems and templates to form a file system that can be 
used as a FreeBSD jail.

In order to use Bastille, the system needs to be prepared. This process
involves creating the  base file systems and a default template. By default, 
Bastille will place all its files under `/usr/local/bastille`. During 
preparation, the following structure will be created: 

```
base/                       | Directory under which to store base file systems
base/ro                     | Read only files common to all jails
base/rw                     | Modifiable files
templates/                  | Directory under which to store templates
templates/default           | Directory for default template
```

Bastille can be made to use a different path for its files. This is useful if, 
for example, you wish to simultaneously maintain base file systems for different 
versions of FreeBSD.  In such a scenario, you would typically prepare 
a directory structure for each release.

## Base file systems
There are two base file system parts: a read-only ("ro") part and a writable one 
("rw").  The read-only part is populated with static files that typically do not 
change between FreeBSD releases (or patch levels), whereas the writable part 
consists of configuration files and directories that are typically changed 
during normal use of a jail.

The separation between these two parts allows the read-only part to be shared
among all jails, thus saving disk space. Additionally, when patching or
upgrading, the read-only part needs to be updated once rather than for each
jail individually.

## Templates
A template is a set of files that are copied over a new jail's file system. That 
is, after a jail has obtained its own copy of the writable base part, the files 
from the template are copied into the new jail's filesystem, overriding any 
existing files. 

The file's relative path within the template's root will be preserved as it is 
cloned into a new jail.

By default, templates contain a master.passwd file that disables password-based 
root logins as well as a copy of the host's `/etc/localtime` and 
`/etc/resolv.conf`' file.

# Jails
A jail is identified by its name and typically resides under 
`/usr/local/bastille/jails/` in a folder sharing its name. Every jail has 
a directory named `ro` directly below its root onto which the read-only base 
file system part is mounted read-only. 

Directories that contain files which typically remain unchanged between FreeBSD 
releases or patch levels, are not created directly under the jail's root file 
system. Instead, symlinks are created to their corresponding directories  under 
`ro`.  Directories which do contain modifiable files are created and populated 
normally. Such directories include  as /etc, /tmp, /usr/local and /usr/obj.  
These modifiable files and directories are copied from the base writable file 
system part preserving ownership, permissions and other attributes.

Every jail is based on a template. If none was explicitly specified, the 
`default` template is used. Files in a template supersede those in the base rw 
part.

# Usage
Bastille provides the `bastille` script which expects at least a command, which 
may appear anywhere on the command line, and some additional options. 

## Synopsis
```
    bastille [-b bastille-dir] [-c config-file] [-m mirror]
        [-r release] prepare
    bastille [-b bastille-dir] [-c config-file] [-m mirror]
        [-r release] add-components
    bastille -n name [-b bastille-dir] create-template
    bastille -n name [-b bastille-dir] [-4 ip4.addr]
        [-6 ip6.addr] [-h hostname] [-t template] create-jail
    bastille [-b bastille-dir] update
    bastille [-b bastille-dir] [-r release] upgrade
    bastille version
```

## Global options
```
Global options:
    -b bastille-dir     -- Directory where to put and look for the jail base 
                           filesystem and templates 
                           (default: /usr/local/bastille)
    -c config-file      -- Store and look for jail definitions in config-file 
                           (default: /etc/jail.conf
    -m mirror           -- Mirror from which to download system components.
    -r release          -- Release level of which to download the system 
                           components (when used with prepare) or to which to 
                           upgrade the jail base filesystem (when used with 
                           upgrade).  Defaults to the release level of the host 
                           system
```

## prepare
Prepares Bastille for use. First, this command creates the directory structure 
described for the base parts and templates under `<bastille-dir>`.  Secondly, it 
downloads the base system component, splits its contents into a read-only and 
writable base file system part and installs the corresponding files in the 
correct base directory. Finally, a default template is created

## add-components
Adds system components to the read-only base parts. This must be a valid system 
component such as src, ports.

### Command options
```
Command options:
    -a component        -- Adds component. This option may be repeated several
                           times.
```

## create-template
Creates a new template. A new template will be created under 
`<bastille-dir>/templates`. By default a template contains an 
`etc/master.passwd` file which disables password-based logins for root plus 
a copy of the host's `/etc/localtime` and `/etc/resolv.conf` file.

### Command options
```
Command options:
    -n name             -- Name of the template to create (Required)
```

## create-jail
Creates a new jail in a directory sharing the jail's name under 
`<bastille-dir>/jails`.

### Command options
```
Command options:
    -n name             -- Name of the new jail (Required)
    -h hostname         -- Hostname of the new jail, defaults to the name of the 
                           jail
    -t template         -- Template on which to base the new jail, defaults to 
                           'default'
    -4 [x.x.x.x[,...]]  -- A list of IPv4 addresses with which to associate the 
                           new jail
    -6 [ipv6[,...]]     -- A list of IPv6 addresses with which to associate the 
                           new jail
```

## update
Updates the base file systems and jails to the latest patch level. 

This command will update the base file system and all jails in 
`<bastille-dir>/jails`. This is done, because once the base file systems are 
updated, its impossible to figure out from which version they were updated.

## upgrade
Upgrades the base file systems and jails to the release specified with the `-r` 
option or, if the `-r` option was omitted, to the host's release.

This command will upgrade the base file system and all jails in 
`<bastille-dir>/jails`. This is done, because once the base file systems are 
updated, its impossible to figure out from which version they were upgraded..

## version
Displays Bastille's version number
