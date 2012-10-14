# zfSnap

zfSnap is a simple sh script to make rolling zfs snapshots with cron. The main
advantage of zfSnap is it's written in 100% pure _/bin/sh_ so it doesn't
require any additional software to run.

zfSnap keeps all information about snapshot in snapshot name.

zfs snapshot names are in the format of **Timestamp--TimeToLive**.

**Timestamp** includes the date and time when the snapshot was created and
**TimeToLive (TTL)** is the amount of time for the snapshot to stay alive
before it's ready for deletion.



## Description of Name Format

**Timestamp** is saved as **year-month-day_hour.minute.second**

  * Example: **2010-08-02_18.45.00**

**TimeToLive** can contain numbers and modifiers and is calculated in seconds

  * Example 1: **1y6m5d2h** - One year, six months, five days, and two hours
  * Example 2: **2m** - Two months
  * Example 3: **216000s** - Two hundred and sixteen thousand seconds (~2 months)
  * Example 4: **216000** - Two hundred and sixteen thousand seconds (the s for seconds is optional)




## Valid TTL Modifiers

  * **y** - years (equals 365 days)
  * **m** - months (equals 30 days)
  * **w** - weeks (equals 7 days)
  * **d** - days
  * **h** - hours
  * **M** - minutes
  * **s** - seconds

**NOTE:** You don't need to include all of these if you are not using them, but
modifiers must be used in this ordering.



# Command line options

**zfSnap** [ _generic options_ ] \[\[ -a _ttl_ ] [ -r|-R ] _zpool/zfs1_ ... ] ...




## Generic options

  * **-d** - deletes snapshots older than the TTL in the snapshot name
  * **-e** - Return exit code return number of failed actions
  * **-F _age_** - Force delete all snapshots older than _**age**_
  * **-o** - Use old timestamp format (**yyyy-mm-dd_hh:mm:ss--ttl**) instead of new (**yyyy-mm-dd_hh.mm.ss--ttl**) (for compatibility with snapshots created before zfSnap v1.4.0)
  * **-s** - Don't do anything on pools running resilver
  * **-S** - Don't do anything on pools running scrub
  * **-z** - round down seconds in the snapshot name to 00 (such as 18:06:15 to 18:06:00)
  * **-n** - perform a test run with no changes made
  * **-v** - verbose output
  * **-zpool28fix** - Workaround for zpool v28 zfs destroy -r bug (See [[Misc info]])

**NOTE:** Generic options must be specified at beginning of command line



## Options

  * **-a _TTL_** - change default **TTL** (if TTL doesn't match TTL pattern, results may vary). Default TTL is 1m
  * **-r** - create **recursive** snapshots for all zfs file systems that follow this switch
  * **-R** - create **non-recursive** snapshots for all zfs file systems that follow this switch
  * **-p _prefix_** - use **prefix** for snapshots after this switch
  * **-P** - **don't use prefix** for snapshots
  * **-D _zpool/dataset_** - delete all zfSnap snapshots of the specified _**zpool/dataset**_ (Ignores TTL)




# Using zfSnap with /etc/crontab

zfSnap was designed to work with ''/etc/crontab''



## Examples of Rolling Snapshots Using Crontab

Hourly recursive snapshots of an entire pool kept for 5 days

	# Minute   Hour   Day of month   Month   Day of week   Who    Command
	5          *      *              *       *             root   /usr/local/sbin/zfSnap -a 5d -r zpool



Snapshots created at 6:45 and 18:45 kept for 2 weeks of different datasets in
different zpools

	45      6,18    *       *       *       root    /usr/local/sbin/zfSnap -a 2w zpool2/git zpool2/jails zpool2/templates -r zpool2/jails/main zpool2/jails/share1 zpool1/local zpool1/var


**NOTE:** You can use **-a**, **-r** and **-R** as much as you want in a single
line

At 2:15 on the first of every month do

  * zpool/var recursive and hold it for 1 year
  * zpool/home and hold it for 6 minutes
  * zpool/usr and hold it for 3 months
  * zpool/root non-recursive and hold it for 3 months

The magic entry
	15      2    1       *       *       root    /usr/local/sbin/zfSnap -a 1y -r zpool/var -a 6M zpool/home -a 3m zpool/usr -R zpool/root




## Delete old snapshots

It is probably better to delete old snapshots once a day (at least on servers),
then adding **-d** switch to every crontab entry.

This is because deleting zfs snapshots is slower than creating them. Also who
cares if few snapshots stay few hours longer?



This crontab entry will delete old zfs snapshots at 1:00

	# Minute   Hour   Day of month   Month   Day of week   Who    Command
	0          1      *              *       *             root   /usr/local/sbin/zfSnap -d



## Delete old snapshots with old timestamp format

	# Minute   Hour   Day of month   Month   Day of week   Who    Command
	0          1      *              *       *             root   /usr/local/sbin/zfSnap -d -o

Note that this only deletes snapshots with old timestamp format. If you need to
delete snapshots with new timestamp format, you need to add another cron job
(without **-o** flag)



## Delete old snapshots with prefixes

If you are creating snapshots with prefix (**-p** flag) and want to delete
these snapshots (**-d**), then you need to run **zfSnap -d** and specify all
prefixes with **-p** flags.

For example:

if you have create snapshots with *test\_* and *test\_me\_* prefixes simply
running

	# zfSnap -d

won't delete these snapshots.

What you need to run is

	# zfSnap -d -p test_ -p test_me

This will delete all old snapshots without prefix, and snapshots with *text\_*
and *test\_me\_* prefixes



## Delete all zfSnap snapshots on specific filesystem

Since zfSnap v1.5.0 you can delete all zfSnap snapshots on specific filsystems



For example you make recursive zfSnap snapshots of your entire zpool,

but you don't want to keep snapshots of /tmp and /var/tmp,

because they obviously eat up space.



For this you can

	# zfSnap -D zpool/tmp -D zpool/var/tmp

this will delete all zfSnap snapshots of zpool/tmp and zpool/var/tmp ignoring
ttl



**NOTE:** -D option will only delete snapshots that match zfSnap snapshot name
pattern (either old version, or new one)



You can also delete all zfSnap snapshots of specific filesytem recursively.

For example:

	# zfSnap -r -D zpool/var

Will delete all zfSnap snapshots of zpool/var and all it's sub-filesystems



#### Sample snapshot names

	$ zfs list -t snapshot | grep var
	zpool/var@2010-08-02_12.06.00--1d           8,57M      -   242M  -
	zpool/var@2010-08-02_13.06.00--1d           7,31M      -   243M  -
	zpool/var@2010-08-02_14.06.00--1d           7,43M      -   243M  -
	zpool/var@2010-08-02_15.06.00--1d           7,56M      -   243M  -
	zpool/var@2010-08-02_16.06.00--1d           7,31M      -   243M  -
	zpool/var@2010-08-02_17.06.00--1d           7,18M      -   243M  -
	zpool/var@2010-08-02_18.45.00--1m           5,08M      -   247M  -
	zpool/var@2010-08-02_19.06.00--1d           1,09M      -   243M  -
	zpool/var@2010-08-02_20.06.00--1d           1,37M      -   243M  -
	zpool/var@2010-08-02_21.06.00--1d           1,62M      -   243M  -
	zpool/var@2010-08-02_22.06.00--1d           1,57M      -   243M  -
	zpool/var@2010-08-02_23.06.00--1d           1,16M      -   243M  -
	zpool/var@2010-08-03_00.06.00--5d           1,28M      -   243M  -
	zpool/var@2010-08-03_01.06.00--5d           1,07M      -   243M  -
	zpool/var@2010-08-03_02.06.00--5d            922K      -   243M  -
	zpool/var@2010-08-03_03.06.00--5d           1,45M      -   242M  -
	zpool/var@2010-08-03_04.06.00--5d            729K      -   242M  -
	zpool/var@2010-08-03_05.06.00--5d            622K      -   241M  -
	zpool/var@2010-08-03_06.06.00--5d            598K      -   241M  -
	zpool/var@2010-08-03_06.45.00--2w           1,34M      -   242M  -
	zpool/var@2010-08-03_07.06.00--5d            662K      -   242M  -
	zpool/var@2010-08-03_08.06.00--5d            847K      -   242M  -
	zpool/var@2010-08-03_09.06.00--5d            837K      -   242M  -
	zpool/var@2010-08-03_10.06.00--5d           1,08M      -   242M  -
	zpool/var@2010-08-03_11.06.00--5d           1,22M      -   243M  -
	zpool/var@2010-08-03_12.06.00--5d            241K      -   243M  -
