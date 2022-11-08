# Sending Bug Reports to the Squid Team

Bug reports for Squid should be registered in our [bug
database](http://bugs.squid-cache.org/).

Any bug report must include

  - The Squid release version

  - Your Operating System type and version

  - A clear description of the bug symptoms.

  - If your Squid crashes the report must include a coredumps stack
    trace as described below

Please note that:

  - bug reports are only processed if they can be reproduced or
    identified in the current STABLE or development versions of Squid.

  - If you are running an older version of Squid the first response will
    be to ask you to upgrade unless the developer who looks at your bug
    report immediately can identify that the bug also exists in the
    current versions.

  - It should also be noted that any patches provided by the Squid
    developer team will be to the current STABLE version even if you run
    an older version.

## crashes and core dumps

There are two conditions under which squid will exit abnormally and
generate a coredump. First, a SIGSEGV or SIGBUS signal will cause Squid
to exit and dump core. Second, many functions include consistency
checks. If one of those checks fail, Squid calls abort() to generate a
core dump.

If you have a core dump file, then use gdb to extract a stack trace from
the core using a "bt full" command (if supported) or a "backtrace"
command (can be typed as "bt" and is always supported, but gives less
information):

    % gdb /usr/local/squid/sbin/squid core
    gdb> backtrace

Many people report that Squid doesn't leave a coredump anywhere. This
may be due to one of the following reasons:

  - Resource Limits
    
      - The shell has limits on the size of a coredump file. You may
        need to increase the limit using ulimit or a similar command
        (see below)

  - Write Permissions
    
      - The system user account for Squid (i.e. proxy, nobody, etc)
        needs write permissions to [coredump destination
        directory](#Coredump_Location)

  - sysctl options
    
      - On FreeBSD, you won't get a coredump from programs that call
        setuid() and/or setgid() (like Squid sometimes does) unless you
        enable this option:

<!-- end list -->

    # sysctl -w kern.sugid_coredump=1

  - No debugging symbols
    
      - The Squid binary must have debugging symbols in order to get a
        meaningful coredump. The debugging traces we need look something
        like this:

<!-- end list -->

    Core was generated by `(squid) -D'.
    Program terminated with signal 6, Aborted.
    
    (gdb) bt
    #0  0x006ad7a2 in _dl_sysinfo_int80 () from /lib/ld-linux.so.2
    #1  0x006ed7a5 in raise () from /lib/tls/libc.so.6
    #2  0x006ef209 in abort () from /lib/tls/libc.so.6
    #3  0x0806b987 in xassert (msg=Could not find the frame base for "xassert".
    ) at debug.c:514
    #4  0x0808170b in httpBuildRequestHeader (request=0x10791f40,
    orig_request=0x10791f40, entry=0xfc6ba30, hdr_out=0xbfed04f0, flags=
          {proxying = 0, keepalive = 1, only_if_cached = 0, keepalive_broken = 0,
    abuse_detected = 0, request_sent = 0, front_end_https = 0, originpeer = 0}) at
    http.c:1195
    ...

if you find the trace contains a lot of lines with **??** and mentions
no symbols found. It is usually useless and you will need to run a
version of Squid where the debug symbols have not been removed.

  - Threads and Linux
    
      - On Linux, threaded applications do not generate core dumps. When
        you use the aufs cache\_dir type, it uses threads and you can't
        get a coredump.

  - It did leave a coredump file, you just can't find it.

## Resource Limits

These limits can usually be changed in shell scripts. The command to
change the resource limits is usually either *ulimit* or *limits*.
Sometimes it is a shell-builtin function, and sometimes it is a regular
program. Also note that you can set resource limits in the
*/etc/login.conf* file on FreeBSD and maybe other systems.

To change the coredumpsize limit you might use a command like:

    limits coredump unlimited

If Squid binary is started by RHEL6/CentOS6 init script, you may need to
set variable *DAEMON\_COREFILE\_LIMIT="unlimited"* in the init script or
the script's configuration file (usually /etc/sysconfig/squid).

For systemd units the equivalent option is *LimitCORE=infinity*.

## Debugging Symbols

To see if your Squid binary has debugging symbols, use this command:

    % nm /usr/local/squid/bin/squid | head

The binary has debugging symbols if you see gobbledegook like this:

    0812abec B AS_tree_head
    080a7540 D AclMatchedName
    080a73fc D ActionTable
    080908a4 r B_BYTES_STR
    080908bc r B_GBYTES_STR
    080908ac r B_KBYTES_STR
    080908b4 r B_MBYTES_STR
    080a7550 D Biggest_FD
    08097c0c R CacheDigestHashFuncCount
    08098f00 r CcAttrs

There are no debugging symbols if you see this instead:

    /usr/local/squid/bin/squid: no symbols

Debugging symbols may have been removed by your install program. If you
look at the squid binary from the source directory, then it might have
the debugging symbols.

## Coredump Location

The core dump file will be left in one of the following locations:

1.  The *coredump\_dir* directory, if you set that option.

2.  The first *cache\_dir* directory if you have used the
    *cache\_effective\_user* option.

3.  The current directory when Squid was started

Recent versions of Squid report their current directory after starting,
so look there first:

    2000/03/14 00:12:36| Set Current Directory to /usr/local/squid/cache

If you cannot find a core file, then either Squid does not have
permission to write in its current directory, or perhaps your shell
limits are preventing the core file from being written.

Often you can get a coredump if you run Squid from the command line like
this (csh shells and clones):

    % limit core un
    % /usr/local/squid/bin/squid -NCd1

Once you have located the core dump file, use a debugger such as *dbx*
or *gdb* to generate a stack trace:

    % gdb /usr/local/squid/sbin/squid core
    gdb> backtrace

If possible, you might keep the coredump file around for a day or two.
It is often helpful if we can ask you to send additional debugger
output, such as the contents of some variables. But please note that a
core file is only useful if paired with the exact same binary as
generated the corefile. If you recompile Squid then any coredumps from
previous versions will be useless unless you have saved the
corresponding Squid binaries, and any attempts to analyze such coredumps
will most certainly give misleading information about the cause to the
crash.

In some environments, coredump is handled by dedicated utility for
management purposes. For example, in RHEL6/CentOS6 environment it may be
*abrtd* with *abrt-addon-ccpp* (man abrtd, man abrt-install-ccpp-hook).
In systemd environments it is *systemd-coredump* (man systemd-coredump).
You can check it by reading */proc/sys/kernel/core\_pattern* (man core):

    # cat /proc/sys/kernel/core_pattern 
    |/usr/libexec/abrt-hook-ccpp %s %c %p %u %g %t e

The core\_pattern for systemd-coredump:

    # cat /proc/sys/kernel/core_pattern 
    |/usr/lib/systemd/systemd-coredump %P %u %g %s %t %c %e

Therefore, you should be familiar with the handlers to control coredump
generation and location. Or you can disable the handler and use usual
methods to handle coredumps if a server is dedicated for Squid. For
example, you can use following methods to disable aforementioned
handlers:

    # chkconfig abrt-ccpp off
    # service abrt-ccpp stop
    # cat /proc/sys/kernel/core_pattern 
    core

To disable systemd-coredump:

    # echo kernel.core_pattern=core > /etc/sysctl.d/50-coredump.conf
    # /usr/lib/systemd/systemd-sysctl --prefix kernel.core_pattern
    # cat /proc/sys/kernel/core_pattern 
    core

## Using gdb debugger on Squid

If you CANNOT get Squid to leave a core file for you then one of the
following approaches can be used

First alternative is to start Squid under the contol of GDB

    % gdb /path/to/squid
    handle SIGPIPE pass nostop noprint
    handle SIGTERM pass nostop noprint
    handle SIGUSR1 pass nostop noprint
    handle SIGHUP  pass
    handle SIGKILL pass
    handle SIGSEGV stop
    handle SIGABRT stop
    run -NYCX
    [wait for crash]
    backtrace
    quit

## Using gdb debugger on a live proxy (with minimal downtime)

The drawback from the above is that it isn't really suitable to run on a
production system as Squid then won't restart automatically if it
crashes. The good news is that it is fully possible to automate the
process above to automatically get the stack trace and then restart
Squid. Here is a short automated script that should work:

    trap "rm -f $$.gdb" 0
    cat <<EOF >$$.gdb
    handle SIGPIPE pass nostop noprint
    handle SIGTERM pass nostop noprint
    handle SIGUSR1 pass nostop noprint
    handle SIGHUP  pass
    handle SIGKILL pass
    handle SIGSEGV stop
    handle SIGABRT stop
    run -NYCd3
    backtrace
    quit
    EOF
    while sleep 2; do
      gdb -x $$.gdb /path/to/squid 2>&1 | tee -a squid.out
    done

Other options if the above cannot be done is to:

1.  Build Squid with the --enable-stacktraces option, if support exists
    for your OS (exists for Linux glibc on Intel, and Solaris with some
    extra libraries which seems rather impossible to find these days..)

2.  Run Squid using the "catchsegv" tool. (Linux glibc Intel)
    
    ℹ️
    these approaches do not by far provide as much details as using gdb.

## Attaching gdb debugger to and already running Squid

First locate the PID number for the particular Squid worker you are
wanting to debug.

    % gdb /path/to/squid
    handle SIGPIPE pass nostop noprint
    handle SIGTERM pass nostop noprint
    handle SIGUSR1 pass nostop noprint
    handle SIGHUP  pass
    handle SIGKILL pass
    handle SIGSEGV stop
    handle SIGABRT stop
    
    attach [worker PID]
    [wait for crash]
    backtrace
    detach
    quit

## Getting the current stack of a live proxy (without downtime)

The following trick may be useful if you suspect that your Squid is
stuck in a busy loop, and/or you want to know what your Squid is doing
"right now", but you do not want to suspect the Squid process.

1.  To dump the current stack using gdb:
    
        sudo gdb -n -batch -ex backtrace -pid <PID>
    
    You may not need/want the `-n` (i.e. do not load gdb initialization
    files) option. Using `'backtrace full'` instead of `backtrace` will
    give even more info but might be a tad slower.

2.  To dump the current stack using pstack(1):
    
        sudo pstack <PID>

3.  To quickly check whether Squid is waiting for a system call:
    
        sudo cat /proc/<PID>/stack
    
    This approach does not show Squid functions leading to a system
    call.

In the above commands, `<PID>` stands for a process ID of the Squid
process you want to debug. It is usually a worker process, but it could
also be another kid process or even the master process.

# Debugging Squid

If you believe you have found a non-fatal bug (such as incorrect HTTP
processing) please send us a section of your cache.log with debugging to
demonstrate the problem. The cache.log file can become very large, so
alternatively, you may want to copy it to an FTP or HTTP server where we
can download it.

Once you have the debugging captured to *cache.log*, take a look at it
yourself and see if you can make sense of the behavior which you see. If
not, please feel free to send your debugging output to the *squid-users*
or *squid-bugs* mailing lists.

## Debugging a single transaction

Unfortunately, it is not yet possible to debug a single transaction, but
the following procedure minimizes logging noise and may help developers
to pinpoint the problem:

1.  Locate your Squid log file or equivalent. In this example, we will
    call it *cache.log*.

2.  Enable detailed (level-7) or full (level-9) debugging. See the
    sections below for details.

3.  Start Squid if necessary.

4.  Run "tail -f cache.log \> partial-cache.log". This will start
    appending new debugging to the *partial-cache.log* file.

5.  Reproduce the failing transaction, using a single request if
    possible. Please note that reloading a page in a browser often sends
    dozens or even hundreds of requests to Squid. Ideally, use
    [squidclient](/SquidClientTool#),
    wget, curl, or another "single-request" tool when possible.

6.  Kill the "tail" command above.

7.  Share the resulting partial-cache.log, compressing it if needed.
    Please note that it may contain sensitive information such as
    passwords.

## Detailed Debug Output

It is easy to get level-7 debugging on a running squid process:

    squid -k debug

The above command sends the running Squid version a signal which causes
many (but not all) debug() statements in the source code to write to the
*cache.log* file or equivalent. Repeating the same command restores the
previous debugging level.

To debug what happens before "squid -k debug" starts working, see the
**-X** command line option discussed below.

## Full Debug Output

To enable full or level-9 debugging (i.e., to force every debugging
statement in Squid to emit some output when reached), you have two
options:

1.  Set debug\_options in squid.conf to ALL,9. Doing so will debug what
    happens after the configuration file is parsed. This is sufficient
    to triage most runtime problems.

2.  Start Squid with the **-X** command line option. Doing so will debug
    what happens both before and after debug\_options in the
    configuration file are parsed.

When started with -X (or -d) command-line option, before Squid opens
cache.log or starts sending debugging to a logging daemon, Squid writes
debugging lines to the standard error stream (stderr). When not started
with those command-line options, very little or no debugging is produced
until after Squid parses the configuration file and starts honoring the
settings configured there.

Unfortunately, it is impossible enable full debugging on a running Squid
process, but "squid -k debug" discussed above will enable level-7
debugging.

## Debug Sections

To enable selective debugging (e.g. for one source file only), you need
to edit **squid.conf** and add to the **debug\_options** line. Every
Squid source file is assigned a debugging **section**. The debugging
section assignments can be found by looking at the top of individual
source files, by reading the file *debug-sections.txt*, or looking at
[KnowledgeBase/DebugSections](/KnowledgeBase/DebugSections#).

## Debug Levels

You also specify the debugging *level* to control the amount of
debugging. Higher levels result in more debugging messages. For example,
to enable full debugging of Access Control functions, you would use:

    debug_options 28,9

Then you have to restart or reconfigure Squid.

# Capturing packets

Sometimes, after inspecting coredump traces and debug output, developers
may ask you to collect packets on both sides of Squid (Squid-to-origin
and Squid-to-client). You can use powerful tool **tcpdump** to
accomplish the operation. Below are presented some common examples and
methods.

## Collecting packets on intercepting proxy

In standard/common configuration, intercepting proxy has two network
interfaces. One facing the Internet and second one facing the local
network. So, to capture Squid-to-origin transactions you have to attach
tcpdump to WAN interface and consequently to LAN interface for
Squid-to-client transactions. You have to be in tcpdump group or be a
root user to capture traffic on network interfaces.

Syntax:

    tcpdump -s 0 -i $nic -w /path/to/dump.pcap port $port and host $origin_ip [and host $client_ip]

where **$nic** is WAN or LAN network interface, **$port** is intercepted
port, **$origin\_ip** is origin server IP, **$client\_ip** is client IP
in local network. The section 'and host $client\_ip' applicable only for
Squid-to-client transactions on NAT proxies. TProxy interception proxies
can use the section for both transaction types. The argument **-s 0**
required to instruct tcpdump to save full packet, **-w** specifies dump
location.

For example, to capture Squid-to-origin transaction on NAT proxy:

    tcpdump -s 0 -i eth1 -w /tmp/squid-to-example.com.pcap port 80 and host 10.11.12.13

To capture Squid-to-origin transaction on TProxy proxy:

    tcpdump -s 0 -i eth1 -w /tmp/squid-to-example.com.pcap port 80 and host 10.11.12.13 and host 172.16.1.100

To capture Squid-to-client transaction:

    tcpdump -s 0 -i eth0 -w /tmp/squid-to-client100.pcap port 80 and host 10.11.12.13 and host 192.168.1.100

To limit amount of collected traffic, immediately after you started both
tcpdump instances, initiate the problem HTTP request. After you have got
desired result, immediately stop both tcpdump instances.

## Collecting packets on explicit proxy

The main difference between the methods is that Squid-to-client
communication is explicit between Squid's IP/proxy port and client IP.
So the syntax is:

    tcpdump -s 0 -i $nic -w /path/to/dump.pcap port $proxy_port and host $proxy_ip and host $client_ip

where **$proxy\_port** and **$proxy\_ip** are values used by user agents
(e.g. browsers) to access Squid. For example:

    tcpdump -s 0 -i eth0 -w /tmp/squid-to-client100.pcap port 3128 and host 192.168.1.1 and host 192.168.1.100

Please note, that the capture would include all traffic to proxy server
from the client. So try to limit the amount of active HTTP sessions on
the client while capturing the traffic.

## Collecting packets leading to Squid's crash

Sometimes Squid may crash due to the processing of packets to/from known
destination, so we have to capture the problem traffic for analysis. To
do so you have an option to capture the traffic for long time until you
encounter a crash. As packet dump can easily become very large for that
operation, so it is better to enable dump file rotation. For example:

    tcpdump -s 0 -i eth0 -G 300 -w /tmp/squid-to-client100-%Y-%m-%d_%H:%M:%S.pcap port 3128 and host 192.168.1.1 and host 192.168.1.100

where **-G** specifies rotation interval in seconds, and the flexible
string like **%Y-%m-%d\_%H:%M:%S** in filename exposes to date (man
strftime) when the rotation occurred.

In above example, tcpdump will rotate the dump file every 5 minutes.
Once you have found that Squid crashed and got exact time of the
failure, you can easily find interesting 5-minutes part of the packet
dump. The target directory for the dumps should be big enough to
accommodate continuous capture. If tcpdump drops its privileges to
ordinary user (usually tcpdump), the user should have write access to
target directory.

If you have size-limited capture storage, you can use rotating buffer
option of tcpdump. It allows to reserve fixed storage size for captures.
For example:

    tcpdump -s 0 -i eth1 -w /tmp/squid-to-example.com.pcap -W 10 -C 100 port 80 and host 10.11.12.13

where **-W** specifies maximum number of dump files and **-C** specifies
maximum size in (MB) for a dump file.

In the above example, tcpdump will rotate a dump file and append suffix
to the filename when it reaches 100MB. It will overwrite the oldest dump
file when it reaches configured maximum number of the files. In this
configuration the maximum capture storage size will be 1GB. The drawback
of the method, is that you have to periodically monitor for Squid's
activity and stop the capture as soon as possible when you detect an
assertion. Otherwise, the interesting file dump would be overwritten.

Back to the
[SquidFaq](/SquidFaq#)