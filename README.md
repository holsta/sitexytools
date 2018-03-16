
# SiteXYTools



## GOALS

1. Completely unattended customisation.
2. ability to install packages as part of OS install.
	Either by shipping package lists to a node telling it which packages
	to install from a particular location; or
	by installing packages into a fake root of the system's
	siteXY-<hostname>.tgz
3. ability to add users as part of the OS install.
	How to integrate into NIS/LDAP/etc? For now, just
	parse a plain text file and add users accordingly.
4. rm and touch arbitrary files
	touch_list and rm_list contain paths to files that need to
	be deleted or touched (created) during install. Handy for
	creating dummy ssh/isakmpd keyfiles on slow machines that
	won't need them anyway.
5. Arbitrary roles for hosts
	For every <hostname>/.role-<rolename> that exists, see if
	../roles/<rolename> exists, and if so, include the contents
	of that directory at the root of the tarball. Maintain one
	copy of, say, DNS data files, but easily copy them to four
	systems.

## Compatibility

siteXYtools 1.x is incompatible with siteXYtools 0.x

## OVERVIEW

The suite of siteXYtools consist of a number of tools for
OpenBSD that aid in managing host-specific configurations, deploying 
such configurations to live hosts and generating siteXY.tgz (e.g.
site40.tgz) files for customized OpenBSD installations.

The tools implement three phases that are interchangable at any time,
meaning the method of generating files or distributing files can be changed
completely without affecting any other part of the suite.

Note however, that while the particular method of generating may
change, the output of the generate process must be understood by the
install process.

We provide a single, standard siteXY.tgz file which fetches and extracts
the host-specific tarball and performs various magic depending on the
contents.

### 1. Generate configuration files

	Tools: sitectl
	Copy complete configuration files based on common files,
	and per-host override files. 
	Minor changes to existing files are implemented by diff/patch.
	
Imagine the following disk layout:

	siteXY-work/
		_siteXYrc
		_roles/
			dns/
			mx/
		common/
			etc/
				ssh/
					authorized_keys/
						holsta
				doas.conf	
				sysctl.conf
				wsconsctl.conf
			patches/
				sshd
		host1/
			etc/
				pf.conf
				bgpd.conf
			patches/
			rm_list
			pkg_add_list
			pkg_path_list
			touch_list
			user_list
		host2/
			_role-mx
		host3/
			_role-dns

In the above 'common' example, all machines managed will receive some
sysctl and wsconsctl config files and a doas.conf file.

It seems holsta's ssh key is stored in etc/ssh/authorized_keys/holsta - and
it's safe to assume that patches/sshd contains a patch against the stock
sshd_config which disables protocol 1, points to /etc/ssh/keys/ for
authorized_keys and more.

Invoking 'generate host1' will cause the script to first of all parse .siteXYrc
in the current directory. If not found, the script will halt. The .siteXYrc
should contain at least a pointer to where final siteXY tarballs are to be
dropped:

	ballDir=/var/www/users/pub/OpenBSD/siteXY

generate will descend into 'host1' and build $ballDir/siteXY-host1.tgz based on its
content. Finally, a master site39.tgz is updated so it contains the latest
contents of 'common' plus install.site and manage scripts. This means that, if
a host fetches just site39.tgz all the common files are installed to that
host.

	$ generate host1
	/tmp/site39-host1.tgz: Configured as tftpd  Updated.
	/tmp/site39.tgz: Including common directory

Invoking 'generate' with no arguments will build a siteXY tarball for each
machine directory, skipping known administrative directory names such as
'roles', 'common', '.svn' and 'CVS'.

	Suggested testbed for making sure we can handle many requirements:
		regular nic, gif0 and carp setup
		httpd.conf
		sshd_config & ssh_host_{dsa,rsa}
		user listing 
		package listing
		crontabs

### 2. Distribute configuration files

	Tool: ssh+rsync, http fetch, ssh+role account.

	For new hosts, build a new siteXY-hostname.tgz for each change so new
		hosts always get the latest change.
	For other hosts, each host may http download or rsync from staging
		area. 

	Distribution may happen as pull via ftp/http/rsync; or
		as push via rsync/ssh role account.

### 3. Install configuration files

	Tool: install.site and manage (sh scripts)

	For new hosts, siteXY.tgz and siteXY-hostname.tgz is downloaded and
		extracted to the root of the drive, the file /install.site is
		executed as part of the standard OpenBSD install. The box is
		rebooted once installation has completed.
	For live hosts, copy files from temporary location to correct
		location with correct permissions, ownership and restart any
		daemons as required. The 'manage' script handles this for
		now.

In both cases, an mtree specification of file paths, owner, group and
permissions is required to enforce a proper permissions policy. For 
usability reasons, configuration files have 'normal' permissions while
being managed. At install time, the permissions, owner and group is set as
required to prevent security leaks, etc.

### RELATED PROJECTS

See Andrew Fresh' sxxu, who took the idea further. https://github.com/afresh1/sxxu
