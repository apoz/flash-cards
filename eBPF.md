
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