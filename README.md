# lab30


Most of the labs in the DTrace course have a description in *lab#.txt* and an executable.
Read the txt file  and run the executable.  At the end of each lab txt file, there is a place
for "Your response".  I'll find a way to get to any/all responses.

These labs should work in both global and nonglobal zones, but not on a VM that does not support DTrace, (e.g., windows, linux bhyve
instances, and other VMs without DTrace support will not work for non-global zone DTrace use).
The labs should work on LX zones, but they'll probably have to be recompiled from source.
Remember that LX system calls are in the `lx-syscall` provider, not the `syscall` provider.

As  part of the training, the instructor will go through some examples that require global zone root access.

The following is a description of lab30 from the DTrace course labs.  It is meant as an example of analysis needed to solve the labs.  If you don't have the lab files, let me know and I'll send the descriptions and binaries.  Most of these labs are simple C programs that are broken in some way.  I'll send the source when you've completed the labs.  



```
# cat  lab30.txt
Min Prereq: Module 3

Customer description:

Application is taking forever to process work. Why might that be?

Your response:
```

First, we'll run ./lab30:

```
# ./lab30
Can't open infile: No such file or directory
```

So the program doesn't take forever.  It fails to work.  Let's first see which system calls return errors when lab30 runs.  Note that I would normally use `truss` for this,
but this is about DTrace...

```
# dtrace -n 'syscall:::return/pid == $target && errno != 0/{printf("%s failed, errno = %d\n", probefunc, errno);}' -c ./lab30
dtrace: description 'syscall:::return' matched 235 probes
Can't open infile: No such file or directory   <-- The error message printed out when running lab30
dtrace: pid 104702 has exited
CPU     ID                    FUNCTION:NAME
0    646                      open:return open failed, errno = 2   <-- The open(2) system call failed.
```

Let's use DTrace to find out what file is trying to be opened.

```

# cat lab30_open.d
#!/usr/sbin/dtrace -s

syscall::open:entry
/execname == "lab30"/
{
    printf("opening %s\n", copyinstr(arg0));
	self->inopen = 1;
}

syscall::open:return
/self->inopen/
{
   printf("open returns %d, errno = %d\n", arg0, errno);
   self->inopen = 0;
}

# chmod +x lab30_open.d
```

On entry to the system calls, the script prints  arg0 (the name of the file being opened).  On return, it prints the return value and the errno.

```
# ./lab30_open.d -c "./lab30"
dtrace: script './lab30_open.d' matched 2 probes
Can't open infile: No such file or directory  <-- the error message from the program
dtrace: pid 131712 has exited
CPU     ID                    FUNCTION:NAME
  0    645                       open:entry opening data100m  <-- lab30 tries to open "data100m"

  0    646                      open:return open returns -1, errno = 2  <-- and fails (grep ENOENT /usr/include/sys/errno.h, or man -s 2 open)

```

So the program expects a file to exist whose name is  data100m.  Many of the labs use this file.  We'll create it to be 100MB large.

```

# mkfile 100m data100m
# ls -l data100m
-rw------T   1 root     root     104857600 Jun  3 16:12 data100m
#
```

And run lab30 again.  We'll run it in the background (alternatively, we can run it with `-c ./lab30`).

```
# ./lab30 &
[1] 132867
#
```

First, let's use `prstat(1)` to see what is doing.

```
# prstat -c
Please wait...
   PID USERNAME  SIZE   RSS STATE  PRI NICE      TIME  CPU PROCESS/NLWP       
132867 root     2052K  804K cpu48    1    0   0:00:21 1.1% lab30/1
    88 root        0K    0K sleep   98  -20   5:03:21 0.1% zpool-zones/268
126332 root       15M   13M sleep    1    0   0:07:28 0.0% sshd/1
126329 root       13M   11M sleep    1    0   0:03:18 0.0% sshd/1
    70 root     2744K 1532K sleep   29    0   0:00:01 0.0% pfexecd/4
  7895 noaccess 2872K 1788K sleep   29    0   0:00:06 0.0% mdnsd/1
   209 root       28M   24M sleep   29    0   0:00:13 0.0% syseventd/23
    21 root     2928K 1628K sleep   29    0   0:00:00 0.0% dlmgmtd/7
  7574 root     4012K  716K sleep    1    0   0:00:17 0.0% ipmon/1
    18 netadm   4244K 2712K sleep   29    0   0:00:00 0.0% ipmgmtd/3
  8018 root     2028K 1360K sleep   59    0   0:00:00 0.0% ttymon/1
  9286 root       63M   52M sleep    1    0   0:01:30 0.0% vmadmd/6
  7063 root     2548K 1004K sleep   22    0   0:02:32 0.0% lldpd/1
    10 root       10M 9112K sleep   29    0   0:02:12 0.0% svc.configd/16
     8 root     8528K 5960K sleep   29    0   0:00:07 0.0% svc.startd/15
Total: 74 processes, 868 lwps, load averages: 0.30, 0.11, 0.07
   PID USERNAME  SIZE   RSS STATE  PRI NICE      TIME  CPU PROCESS/NLWP       
132867 root     2052K  804K cpu48    1    0   0:00:26 1.3% lab30/1
    88 root        0K    0K sleep   98  -20   5:03:21 0.1% zpool-zones/268
126332 root       15M   13M sleep    1    0   0:07:28 0.0% sshd/1
126329 root       13M   11M sleep    1    0   0:03:18 0.0% sshd/1
    70 root     2744K 1532K sleep   29    0   0:00:01 0.0% pfexecd/4
  7895 noaccess 2872K 1788K sleep   29    0   0:00:06 0.0% mdnsd/1
   209 root       28M   24M sleep   29    0   0:00:13 0.0% syseventd/23
    21 root     2928K 1628K sleep   29    0   0:00:00 0.0% dlmgmtd/7
  7574 root     4012K  716K sleep    1    0   0:00:17 0.0% ipmon/1
    18 netadm   4244K 2712K sleep   29    0   0:00:00 0.0% ipmgmtd/3
  8018 root     2028K 1360K sleep   59    0   0:00:00 0.0% ttymon/1
  9286 root       63M   52M sleep    1    0   0:01:30 0.0% vmadmd/6
  7063 root     2548K 1004K sleep   22    0   0:02:32 0.0% lldpd/1
    10 root       10M 9112K sleep   29    0   0:02:12 0.0% svc.configd/16
     8 root     8528K 5960K sleep   29    0   0:00:07 0.0% svc.startd/15
	 Total: 74 processes, 868 lwps, load averages: 0.37, 0.13, 0.08
```

So lab30 is using ~1.3% of the available cpus.  This machine has 56 cores (`psrinfo` output will tell you for your machine), so that is roughly 70% of one core.
Using `prstat` to take a closer look.

```
# prstat -mcLvp 132867
Please wait...
   PID USERNAME USR SYS TRP TFL DFL LCK SLP LAT VCX ICX SCL SIG PROCESS/LWP   
132867 root      73  27 0.0 0.0 0.0 0.0 0.0 0.0   0 455 .70   0 lab30/1
Total: 1 processes, 1 lwps, load averages: 1.07, 0.95, 0.56
   PID USERNAME USR SYS TRP TFL DFL LCK SLP LAT VCX ICX SCL SIG PROCESS/LWP   
132867 root      73  27 0.0 0.0 0.0 0.0 0.0 0.0   0 454 .70   0 lab30/1
Total: 1 processes, 1 lwps, load averages: 1.07, 0.95, 0.57
...
```

So lab30 is spending 73% of its time running user level code (code that makes up lab30), and 27% of the time running in system mode (most likely, handling system calls).
Let's see what system calls lab30 is making.  We'll look at the user side in a later session of the course.

```
# dtrace -n 'syscall:::entry/pid == $target/{@[probefunc] = count();} tick-10s{exit(0);}' -p 132867
dtrace: description 'syscall:::entry' matched 236 probes
CPU     ID                    FUNCTION:NAME
  1  86843                        :tick-10s 

read                                                        9564857
```

In 10 seconds, lab30 did 9564857 read system calls.  Let's see what file(s) are being read.

```
# dtrace -n 'syscall::read:entry/execname == "lab30"/{@[fds[arg0].fi_pathname] = count();} tick-10s{exit(0);}'
dtrace: description 'syscall::read:entry' matched 2 probes
CPU     ID                    FUNCTION:NAME
 32  86843                        :tick-10s 

/var/tmp/max/data100m                                       5961350
#
```

Always the `data100m` file.  Let's check return values.

```
# dtrace -n 'syscall::read:return/pid == $target/{@[arg1, errno] = count();} tick-10s{exit(0);}' -p 132867
dtrace: description 'syscall::read:return' matched 2 probes
CPU     ID                    FUNCTION:NAME
 30  86843                        :tick-10s 

0        0          9685394
#
```

So `read(2)` always returns 0, with errno equal to 0 (i.e., no errors).  Let's check the number of bytes requested on the read call.  This is the 3rd argument to `read(2)`.

```
]# dtrace -n 'syscall::read:entry/pid == 132867/{@["read size: ", arg2] = count();} tick-10s{exit(0);}'
dtrace: description 'syscall::read:entry' matched 2 probes
CPU     ID                    FUNCTION:NAME
 14  86843                        :tick-10s 

read size:                                                        0          8535882
#
```

So `read` is being called with 0 bytes requested.

This is most likely due to a bug in the code.  Possibly a variable is not being initialized correctly.  You should want to look at the code for `read(2)` system calls.

That's an example.  Please do not use `truss` to look at these problems, as truss will often point you in the right direction (i.e., you can probably solve most of these using `truss`, and not learn any DTrace.  Good luck!




