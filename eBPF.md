
eBPF (extended Berkeley Packet Filtering)
=========================================

## Architecture
BPF is a highly advanced VM, running code instructions in an isolated environment.

## BPF programs

The most common way to write BPF programs is by using a subset of C compiled with LLVM. LLVM is a general-purpose compiler that can emit different types of bytecode. In this case, LLVM will output BPF assembly code that we will load into the kernel later. 

````c
#include <linux/bpf.h>
#define SEC(NAME) __attribute__((section(NAME), used))

SEC("tracepoint/syscalls/sys_enter_execve")
int bpf_prog(void *ctx) {
    char msg[] = "Hello, BPF World!";
    bpf_trace_printk(msg, sizeof(msg));
    return 0;
}

char _license[] SEC("license") = "GPL";
````

Details:
* The SEC attribute is used the inform the BPF VM when we want to run this command. In this example, when a rtacepoint in an execve system call is detected.
* We’re using bpf_trace_printk to print a message in the kernel tracing log; you can find this log in /sys/kernel/debug/tracing/trace_pipe.
* To compile the code into bpf bytecode we'd use the following command:
````
clang -O2 -target bpf -c bpf_program.c -o bpf_program.o
````

To load a bpf program into the kernel we can use a helper. Usually the bpf programs die when the process that loaded them dies, but they can be made persistent.

### Types of BPF programs

BPF programs can be divided in 2 categories:
* *Tracing*: Help you better understand what’s happening in your system. They give you direct information about the behavior of your system and the hardware it’s running on. They can access memory regions related to specific programs, and extract execution traces from run‐ ning processes. They also give you direct access to the resources allocated for each specific process, from file descriptors to CPU and memory usage.
* *Networking*: These types of programs allow you to inspect and manipulate the network traffic in your system. They let you filter packets coming from the network interface, or even reject those packets completely. 

Ejemplos de tipos de programas:
* *BPF_PROG_TYPE_SOCKET_FILTER*: allows yo to get -read- access to all the packets managed in one socket.
* *Kprobe programs* are programs that can be used as kprobe handlers. When you write a BPF program that’s attached to a kprobe, you need to decide whether it will be executed as the first instruction in the function call or when the call completes.
* *Tracepoint programs*: This type of program allows you to attach BPF programs to the tracepoint handler provided by the kernel. All tracepoints in your system are defined within the directory /sys/kernel/debug/trac‐ ing/events. There you’ll find each subsystem that includes any tracepoints and that you can attach a BPF program to.
* *XDP programs*: XDP programs allow you to write code that’s executed very early on when a network packet arrives at the kernel. It exposes only a limited set of information from the packet given that the kernel has not had much time to process the information itself. 
* *Perf event programs*: These types of BPF programs allow you to attach your BPF code to Perf events. 
* *Cgroup socket programs*: These types of programs allow you to attach BPF logic to control groups (cgroups). They allow cgroups to control network traffic within the processes that they contain. With these programs, you can decide what to do with a network packet before it’s delivered to a process in the cgroup.
* *Cgroup open socket programs*: These types of programs allow you to execute code when any process in a cgroup opens a network socket.
* *Socket option programs*: These types of programs allow you to modify socket connection options at runtime, while a packet transits through several stages in the kernel’s networking stack.
* *Socket map programs*: give you access to socket maps and socket redirects. 
* *cgroup device programs*: This type of program allows you to decide whether an operation within a cgroup can be executed for a given device.
* *Socket Message Delivery Programs*: These types of programs let you control whether a message sent to a socket should be delivered.
* *Raw Tracepoint Programs*: We talked earlier about a type of program that accesses tracepoints in the kernel. The kernel developers added a new tracepoint program to address the need of accessing the tracepoint arguments in the raw format held by the kernel.
* *Cgroup Socket Address Programs*: This type of program allows you to manipulate the IP addresses and port numbers that user-space programs are attached to when they are controlled by specific cgroups. 
* *Socket Reuseport Programs*: SO_REUSEPORT is an option in the kernel that allows multiple processes in the same host to be bound to the same port. This option allows higher performance in accepted network connections when you want to distribute load across multiple threads.
* *Flow Dissection Programs*: The flow dissector is a component of the kernel that keeps track of the different layers that a network packet needs to go through, from when it arrives to your system to when it’s delivered to a user-space program.


# BPF Maps
BPF maps are key/value stores that reside in the kernel. They can be accessed by any BPF program that knows about them. Programs that run in user-space can also access these maps by using file descriptors. You can store any kind of data in a map, as long as you specify the data size correctly beforehand. The kernel treats keys and values as binary blobs, and it doesn’t care about what you keep in a map.


XDP (Express Data Path)
=======================


# BCC, BPFTRACE and IOVISOR

Some front-ends have been developed that provide higher level languages to create bpf programs.

## BCC (BPF Compiler Collection)

BCC was the first higher-level tracing framework developed for BPF. It provides a C programming environment for writing kernel BPF code and other languages for the user-level interface: python, lua and c++. It's alse the origin of libbcc and current libbpf libraries.

The BCC repo contains >70 BPF tools for performance analysis and troubleshooting.
BCC is better siuted for complex scripts and daemons making use of other libraries.

## Bpftrace

It's a newer front-end that provides a special-purpose, high-level language to develop bpf tools. Bpftrace is built on top of libbcc and libbpf libraries.
Ideal for powerful one-liners and custom short scripts.

## IO Visor
Both BCC and bpftrace do not live in kernel codebase but in a Linux Foundation project called [IO Visor](https://github.com/iovisor).

# BCC Tools
## execsnoop
This tools prints a one line summary of a process when it starts (it works tracing execve system call).
Useful to detect short lived processes that may escape top...
* With *-t* option it prints a timestamp in the line.

## biolatency
Summarizes block device I/O times as a latency histogram.
* When Control-C is typed, the summary is written.
* option *-m*.

## opensnoop
traces file open functions in the kernel
* *-h*: help, many options

# Dynamic instrumentation: Kprobes and Uprobes

# Viewing BPF Instructions: 
## bpftool

Tool to manage bpf programs,etc.
It operates over objects: “{ prog | map | cgroup | perf | net | feature | btf }”

perf and prog objects can be used to find and print tracing programs
*bpftool perf* subcommand shows BPF programms attached via perf_event_open()
It's the ability to insert instrumentation points to live software (in production). It's used by the tools to instrument the start and end of kernel or user functions.
*bpftool prog show* subcommand list all bpf programs (not only the perf_event_open()-based)
*bpftool prog dump xlated* prints the bpf instructions of one program translated into assembly.
*bpftool prog dump jited* shows the machine code for the processor that is executed.
*bpftool btf*: dumps BTF IDs (BTF allows bpftool to include source code lines of a bpf program).

## bpftrace

You can check the bpf instructions with *bpftrace -v*

# Static instrumentation: tracepoints and usdt

USDT User statically defined tracing

# A first look at BPFTrace
There's a tracepoint for open kernel function called syscalls:sys_enter_open.

“bpftrace -e 'tracepoint:syscalls:sys_enter_open { printf("%s %s\n", comm,
    str(args->filename)); }'

You can list all the tracepoints of one type using a wildcard:
“ bpftrace -l 'tracepoint:syscalls:sys_enter_open*'
tracepoint:syscalls:sys_enter_open_by_handle_at
tracepoint:syscalls:sys_enter_open
tracepoint:syscalls:sys_enter_openat”

“bpftrace -e 'tracepoint:syscalls:sys_enter_open* { @[probe] = count(); }'
Attaching 3 probes...
^C

@[tracepoint:syscalls:sys_enter_open]: 5
@[tracepoint:syscalls:sys_enter_openat]: 308”

There's a bpftrace tool called *opensnoop.bt* that does this.


# BPF sysfs interface

Linux 4.4 introduced commands to expose BPF programs and maps via a virtual filesystem, mounted on /sys/fs/bpf.
This allows the creation of BPF programs that are persistent (~daemons) that continue running after the process that loaded them has exited.

# BTF (BPF TYPE FORMAT)
It's a metadata format that encodes debugging information, used to describe BPF programs, BPF maps and much more.
Still in development(2020).


# Performance Analysis

## Linux 60 second analysis

1. uptime
2. dmesg | tail
3. vmstat 1
4. mpstat -P ALL 1
5. pidstat 1
6. iostat -xz 1
7. free -m
8. sar -n DEV 1
9. sar -n TCP,ETCP 1
10. top

### 1. Uptime
Quick way to view the load averages, which indicate the number of tasks (processes) waiting to run. In Linux these numbers include processes wanting to run on the CPUs as well as processes blocked in uninterruptible I/O.
The three numbers are exponentially damped moving sum averages within a 1-min, 5-minute and 15-minute constant. It gives us some idea of how load is changing over time.

### 2. dmesg | tail
This views the last 10 system messages.

### 3. vmstat 1
Virtual memory statistics tool. Whith 1 param prints 1 second summaries (first line is summary since boot).
Columns to check:
* **r**: Number of processes running on a cpu and waiting for a turn. r value greater than CPU count means saturation.
* **free**: Free memory in KB.
* **si, so**: Swap-ins and swap-outs. If these are non-zero you're out of memory.
* **us,sy, id, wa, st**: These are breackdowns on CPU time, on avg across all cpus.

### 4. mpstat -P ALL 1
Prints per CPU time broken down into states.
Look out for high %iowait time (which can be explored with disk I/O tools), and high %sys time (which can be explored with syscall and kernel trancing, and CPU profiling).

### 5. pidstat
Pidstat shows CPU usage per process.

### 6. iostat -xz 1
Shows storage device I/O metrics.
Columns to check:
* **r/s,w/s,rkB/s,wkB/s**: delivered reads, writes, read Kbytes and write Kbytes per second to the device. Use these for workload characterization.
* **await**: The average time for the I/O in miliseconds.
* **avgqu-sz**: The average number of requests issued to the device. Value >1 can be evidence of saturation of device.
* **%util**: Device utilization. This is really a busy percent, showing the time each second the device was doing work.

### 7. free -m
This shows available memory in Mbytes.

### 8. sar -n DEV 1
Network device metrics, interface throughput.

### 9. sar -n TCP,ETCP 1
TCP metrics and TCP errors.
Columns to check:
* **active/s**: Number of locally-initiated TCP connections per second.
* **passive**: Number of remotelly-initiated TCP connections per second.
* **retrans/s**: Number of TCP retransmits per second.

### 10. top
Double check the things with top.


# BCC tool checklist

## Installation

````
root@ubuntu:/home/sysadmin# cat /etc/apt/sources.list.d/iovisor.list
deb [trusted=yes] https://repo.iovisor.org/apt/bionic bionic-nightly main
sudo apt-get update
sudo apt-get install binutils bcc bcc-tools libbcc-examples python-bcc
cd /usr/share/bcc/tools/
````

## 1. execsnoop

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/execsnoop
PCOMM            PID    PPID   RET ARGS
sshd             28855  25666    0 /usr/sbin/sshd -D -R
sh               28857  28855    0
env              28858  28857    0 /usr/bin/env -i PATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin run-parts --lsbsysinit /etc/update-motd.d
run-parts        28858  28857    0 /bin/run-parts --lsbsysinit /etc/update-motd.d
00-header        28859  28858    0 /etc/update-motd.d/00-header
uname            28860  28859    0 /bin/uname -o
uname            28861  28859    0 /bin/uname -r
uname            28862  28859    0 /bin/uname -m
```

execsnoop shows new process execution, by printing one line of output for every execve(2) syscall. Check for short-lived processes. These can consume CPU resources, but may not show up in most monitoring tools that periodically take snapshots of which processes are running.

## 2. opensnoop

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/opensnoop
PID    COMM               FD ERR PATH
28928  vi                  3   0 /etc/ld.so.cache
28927  opensnoop          -1   2 /usr/lib/python2.7/encodings/ascii.x86_64-linux-gnu.so
28927  opensnoop          -1   2 /usr/lib/python2.7/encodings/ascii.so
28927  opensnoop          -1   2 /usr/lib/python2.7/encodings/asciimodule.so
28927  opensnoop          11   0 /usr/lib/python2.7/encodings/ascii.py
28927  opensnoop          12   0 /usr/lib/python2.7/encodings/ascii.pyc
28928  vi                  3   0 /lib/x86_64-linux-gnu/libm.so.6
28928  vi                  3   0 /lib/x86_64-linux-gnu/libtinfo.so.5
28928  vi                  3   0 /lib/x86_64-linux-gnu/libselinux.so.1
28928  vi                  3   0 /lib/x86_64-linux-gnu/libacl.so.1
28928  vi                  3   0 /usr/lib/x86_64-linux-gnu/libgpm.so.2
28928  vi                  3   0 /lib/x86_64-linux-gnu/libdl.so.2
28928  vi                  3   0 /usr/lib/x86_64-linux-gnu/libpython3.6m.so.1.0
```

opensnoop prints one line of output for each open syscall (and its variants), including details of the path that was opened, and whether it was successful (the “ERR” error column). Opened files can tell you a lot about how applications work: identifying their data files, config files, and log files. Sometimes applications can misbehave, and perform poorly, when they are constantly attempting to read files that do not exist.

## 3. ext4slower (or btrfs*, xfs*, zfs*)

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/ext4slower
Tracing ext4 operations slower than 10 ms
TIME     COMM           PID    T BYTES   OFF_KB   LAT(ms) FILENAME
11:36:01 vi             29007  S 0       0          31.22 test.txt
11:36:15 vi             29008  S 0       0          15.63 test2.txt
```
ext4slower traces common operations from the ext4 file system (reads, writes, opens, and syncs), and prints those that exceed a time threshold. This can identify or exonerate one type of performance issue: an application waiting on slow individual disk I/O via the file system.

## 4. biolatency

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/biolatency -m
Tracing block device I/O... Hit Ctrl-C to end.
^C
     msecs               : count     distribution
         0 -> 1          : 8        |****************************************|
         2 -> 3          : 3        |***************                         |
         4 -> 7          : 1        |*****                                   |
```

Biolatency traces disk I/O latency, the time from device issue to completion, and shows this as a histogram. This better explains disk I/O performance than the averages shown by iostat. 
Multiple modes can be examined: modes are values that are more frequent than others in a distribution.

## 5. biosnoop

````
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/biosnoop
TIME(s)     COMM           PID    DISK    T SECTOR     BYTES  LAT(ms)
0.000000    ?              0              R 0          8         4.04
2.023791    ?              0              R 0          8        11.83
3.740882    jbd2/sda1-8    184    sda     W 2694656    12288     6.72
3.746598    jbd2/sda1-8    184    sda     W 2694680    4096      0.37
4.028306    ?              0              R 0          8         0.36
6.044260    ?              0              R 0          8         0.31
8.060196    ?              0              R 0          8         0.30
9.313847    jbd2/sda1-8    184    sda     W 2694688    36864     5.81
9.316481    jbd2/sda1-8    184    sda     W 2694760    4096      0.48
````

biosnoop prints a line of output for each disk I/O, with details including latency. This allows you to examine disk I/O in more detail, and look for time-ordered patterns (e.g., reads queueing behind writes).

## 6. cachestat

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/cachestat
    HITS   MISSES  DIRTIES HITRATIO   BUFFERS_MB  CACHED_MB
       0        0        0    0.00%           95       1063
       0        0        0    0.00%           95       1063
       0        0        0    0.00%           95       1063
     349        0        1  100.00%           95       1063
       0        0        0    0.00%           95       1063
       0        0       10    0.00%           95       1063
    1693        1       11   99.94%           95       1063
```

cachestat prints a one-line summary every second (or every custom interval) showing statistics from the file system cache. Use this to identify a low cache hit ratio, and a high rate of misses. This may gives you a lead for performance tuning.

## 7. tcpconnect

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/tcpconnect
PID    COMM         IP SADDR            DADDR            DPORT
29112  wget         4  192.168.83.93    172.217.168.174  80
29112  wget         4  192.168.83.93    172.217.17.4     80
29113  wget         4  192.168.83.93    172.217.168.174  80
29113  wget         4  192.168.83.93    172.217.17.4     80
29114  wget         4  192.168.83.93    172.217.168.174  80
29114  wget         4  192.168.83.93    172.217.17.4     80
```

tcpconnect prints one line of output for every active TCP connection (e.g., via connect()), with details including source and destination addresses. Look for unexpected connections that may point to inefficiencies in application configuration, or an intruder.

## 8. tcpaccept

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/tcpaccept
PID     COMM         IP RADDR            RPORT LADDR            LPORT
25666   sshd         4  10.95.43.140     57462 192.168.83.93    22
25666   sshd         4  10.95.43.140     57465 192.168.83.93    22
```
tcpaccept is a companion tool to tcpconnect. It prints one line of output for every passive TCP connection (eg, via accept()), with details including source and destination addresses.

## 9. tcpretrans

```
/usr/share/bcc/tools/tcpretrans
Tracing retransmits ... Hit Ctrl-C to end
TIME     PID    IP LADDR:LPORT          T> RADDR:RPORT          STATE
```

tcprerans prints one line of output for every TCP retransmit packet, with details including source and destination addresses, and the kernel state of the TCP connection. TCP retransmissions cause latency and throughput issues. For retransmissions where the TCP session state is ESTABLISHED, look for problems with external networks. For the SYN_SENT state, this may point to target kernel CPU saturation and kernel packet drops as well.

## 10. runqlat

````
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/runqlat
Tracing run queue latency... Hit Ctrl-C to end.

     usecs               : count     distribution
         0 -> 1          : 110      |****                                    |
         2 -> 3          : 459      |********************                    |
         4 -> 7          : 906      |****************************************|
         8 -> 15         : 478      |*********************                   |
        16 -> 31         : 667      |*****************************           |
        32 -> 63         : 267      |***********                             |^C
        64 -> 127        : 101      |****                                    |
       128 -> 255        : 30       |*                                       |
       256 -> 511        : 17       |                                        |
       512 -> 1023       : 5        |                                        |
      1024 -> 2047       : 4        |                                        |
      2048 -> 4095       : 4        |                                        |
      4096 -> 8191       : 2        |                                        |
````

runqlat times how long threads were waiting for their turn on CPU, and prints this time as a histogram. Longer-than-expected waits for CPU access, which threads can suffer due to CPU saturation, misconfigurations, or scheduler issues, can be identified using this tool.

## 11. profile

```
root@ubuntu:/home/sysadmin# /usr/share/bcc/tools/profile
Sampling at 49 Hertz of all threads by user + kernel stack... Hit Ctrl-C to end.
^C
    finish_task_switch
    finish_task_switch
    __schedule
    schedule
    schedule_hrtimeout_range_clock
    schedule_hrtimeout_range
    ep_poll
    sys_epoll_pwait
    do_syscall_64
    entry_SYSCALL_64_after_hwframe
WARNING: Couldn't find .text section in /usr/sbin/odhcpd
WARNING: BCC can't handle sym look ups for /usr/sbin/odhcpdWARNING: Couldn't find .text section in /lib/libgcc_s.so.1
WARNING: BCC can't handle sym look ups for /lib/libgcc_s.so.1WARNING: Couldn't find .text section in /lib/libubus.so
WARNING: BCC can't handle sym look ups for /lib/libubus.soWARNING: Couldn't find .text section in /usr/lib/libnl-tiny.so
WARNING: BCC can't handle sym look ups for /usr/lib/libnl-tiny.soWARNING: Couldn't find .text section in /lib/libuci.so
WARNING: BCC can't handle sym look ups for /lib/libuci.soWARNING: Couldn't find .text section in /lib/libubox.so
WARNING: BCC can't handle sym look ups for /lib/libubox.soWARNING: Couldn't find .text section in /lib/libc.so
WARNING: BCC can't handle sym look ups for /lib/libc.so    [unknown]
    -                odhcpd (10914)
        1

    [unknown]
    -                vim (29406)
        1

```
profile is a CPU profiler, a tool to understand which code paths are consuming CPU resources. It takes samples of stack traces at timed intervals, and prints a summary of unique stack traces and a count of their occurrence.





# BCC
The BPF Compiler Collection (BCC or sometimes called bcc) is an opensource project containing a compiler framework and libraries for building BPF software. It's the main front-end for BPF. Over 70 ready-to-run BPF perf. analysis and troubleshooting tools.

