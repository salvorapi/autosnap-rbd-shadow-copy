autosnap-rbd-shadow-copy
========================

This script make snapshot on Ceph RBD device and mount them for use with Samba vfs shadow_copy2.


## How to use

For test, you need to have a Ceph cluster running and Samba installed on client.

Verify admin access to the ceph cluster : (should not return error)

	$ rbd ls

### Installation

Get this script :

	$ mkdir -p /etc/ceph/scripts/
	$ cd /etc/ceph/scripts/
	$ wget https://raw.github.com/ksperis/autosnap-rbd-shadow-copy/master/autosnap.conf
	$ wget https://raw.github.com/ksperis/autosnap-rbd-shadow-copy/master/autosnap.sh
	$ chmod +x autosnap.sh

### Prepare rbd block device

Create a block device :

	$ rbd create myshare --size=1024
	$ echo "myshare" >> /etc/ceph/rbdmap
	$ /etc/init.d/rbdmap reload
	[ ok ] Starting RBD Mapping: rbd/myshare.
	[ ok ] Mounting all filesystems...done.

Format the block device :

	$ mkfs.xfs /dev/rbd/rbd/myshare
	log stripe unit (4194304 bytes) is too large (maximum is 256KiB)
	log stripe unit adjusted to 32KiB
	meta-data=/dev/rbd/rbd/myshare   isize=256    agcount=9, agsize=31744 blks
	         =                       sectsz=512   attr=2, projid32bit=0
	data     =                       bsize=4096   blocks=262144, imaxpct=25
	         =                       sunit=1024   swidth=1024 blks
	naming   =version 2              bsize=4096   ascii-ci=0
	log      =internal log           bsize=4096   blocks=2560, version=2
	         =                       sectsz=512   sunit=8 blks, lazy-count=1
	realtime =none                   extsz=4096   blocks=0, rtextents=0

Mount the share

	$ mkdir /myshare
	$ echo "/dev/rbd/rbd/myshare /myshare xfs defaults 0 0" >> /etc/fstab
	$ mount /myshare

### Samba config

Add this section in your /etc/samba/smb.conf :

	[myshare]
	        path = /myshare
	        writable = yes
		vfs objects = shadow_copy2
		shadow:snapdir = .snapshots
		shadow:sort = desc

Reload samba

	$ /etc/init.d/samba reload

### Use

Create snapshot directory and run the script :

	$ mkdir -p /myshare/.snapshots
	$ /etc/ceph/scripts/autosnap.sh
	* Create snapshot for myshare: @GMT-2013.08.09-10.16.10-autosnap
	synced, no cache, snapshot created.
	* Shadow Copy to mount for rbd/myshare :
	GMT-2013.08.09-10.14.44

Verify that the first snapshot is correctly mount :

	$ mount | grep myshare
	/dev/rbd1 on /myshare type xfs (rw,relatime,attr2,inode64,sunit=8192,swidth=8192,noquota)
	/dev/rbd2 on /myshare/.snapshots/@GMT-2013.08.09-10.14.44 type xfs (ro,relatime,nouuid,norecovery,attr2,inode64,sunit=8192,swidth=8192,noquota)

Also, you can add this on crontab to run everyday the script :

	$ echo "00 0    * * *   root    /bin/bash /etc/ceph/scripts/autosnap.sh" >> /etc/crontab


## More

Ceph Project : http://ceph.com/

Samba Shadow Copy : http://www.samba.org/samba/docs/man/manpages/vfs_shadow_copy2.8.html


