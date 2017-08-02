
====================
Qubes' requirements:

https://www.qubes-os.org/attachment/wiki/QubesArchitecture/arch-spec-0.3.pdf

- paravirt...

In KVM, the kernel actually becomes the hypervisor for all  the VM processes.
Hypervisor is free to use the whole Linux kernel infrastructure, with  all its drivers and internal interfaces. 
Blurry line between  what code in  the Linux kernel is, and what is  not, used for handling various VM-generated events. Where does the hypervisor start and end? we have to count the whole kernel. Very big, ~4mil LOC.

In  Xen, Everything  is contained within  the hypervisor. At no point does  the execution  path  jump out of the hypervisor to e.g. Dom0.
Easier to perform security code audit of Xen, as it's clear  which  code really belongs  to  the hypervisor.

In Xen, possible to move drivers  and driver backends  out of Dom0. 
- IO Device Emulator (ioemu) moves out of Dom0.
- PV guests in Xen allow to get rid of the I/O Emulator entirely. 
- I/O emulator only needed to run HVM guests, e.g. Windows 
- “stub-domain” mechanism introduced in Xen 3.3, allows to host the I/O emulator in a special per-HVM machine. 
- Only (VM-accessible) element left in  Dom0  is XenStore daemon

Isolating the IOemu (qemu)
- KVM uses standard Linux security mechanisms to isolate and contain  the I/O  emulator process (address  space isolation, standard  ACL  mechanisms). + SELinux 
sandboxing. But, not great.
- Xen uses para-virtualization for the I/O emulator isolation. Better.

For KVM, virtio allows paravirt for *device* emulation. But, no paravirt for CPU stuff? Still need qemu?

Major problems with KVM:
- Doesn't support driver domains
- Doesn't support PV domains

We want to isolate the IOemu in a virt env and per-domain.

Summary:
- Xen  hypervisor is  very small  compared to Linux  kernel 
- Xen allows to move most of the “world-facing” code out of Dom0, including the I/O  emulator, networking code and many  drivers
- Xen supports driver  domains, KVM doesn't
- Xen has better isolation for the I/O emulator process, via two mechanism:
  (1) PV domains just don't use it, so no IOemu at all
  (2) for each HVM, per-domain virtualized-isolated slices of IOemu 

Desired features in a Redox-based hypervisor:
- PV domains, allowing to forgo IOemu (e.g. qemu)
- Stub domains/slices for per-VM hypervisor-enforced isolation of IOemu
- Driver domains

===========
Replacing Xen in Qubes: starting criteria

https://www.qubes-os.org/doc/user-faq/#why-does-qubes-use-xen-instead-of-kvm-or-some-other-hypervisor

What about this other/new (micro)kernel/hypervisor?

Whenever starting a discussion about another (micro)kernel or hypervisor in relation to Qubes, we strongly suggest including answers to the following questions first:

1    What kinds of containers does it use for isolation? Processes? PV VMs? Fully virtualized VMs (HVMs)? And what underlying h/w technology is used (ring0/3, VT-x)?
2    Does it require specially written/built applications (e.g. patched Firefox)?
3    Does it require custom drivers, or can it use Linux/Windows ones?
4    Does it support VT-d, and does it allow for the creation of untrusted driver domains?
5    Does it support S3 sleep?
6    Does it work on multiple CPUs/Chipsets?
7    What are the performance costs, more or less? (e.g. “XYZ prevents concurrent execution of two domains/processes on shared cores of a single processor”, etc.)
8    Other special features? E.g. eliminates cooperative covert channels between VMs?

Here are the answers for Xen 4.1 (which we use as of 2014-04-28):

1    PV and HVM Virtual Machines (ring0/3 for PV domains, VT-x/AMD-v for HVMs).
2    Runs unmodified usermode apps (binaries).
3    Runs unmodified Linux drivers (dom0 and driver domains). PV VMs require special written pvdrivers.
4    Full VT-d support including untrusted driver domains.
5    S3 sleep supported well.
6    Works on most modern CPUs/Chipsets.
7    Biggest performance hit on disk operations (especially in Qubes when complex 2-layer mapping used for Linux qubes). No GPU virtualization.
8    Mostly WorksTM :)


KVM: 

1) Has HVMs and some PV support. Needs better PV support.
2) No, unmodified usermode binaries.
3) Runs unmodified drivers. Also special paravirt drivers.
4) Yes, supports VT-d. Untrusted driver domains? Dunno, maybe not.
5) S3 sleep? Yes.
6) Yes, works on multiple chipsets.
7) Similar to Xen on most benchmarks?
8) Other special features...? Dunno. Eliminates cooperative covert channels between VMs? Dunno.

==========
Qubes critique of Xen security process

https://groups.google.com/forum/#!topic/qubes-devel/uVYowLAXd0M



==========
Nexen

https://www.internetsociety.org/sites/default/files/ndss2017_02A-4_Shi_paper.pdf

Secure Monitor (+ Gate Keeper)
Shared service
Xen slice
Guest VM

SecMon:
- control of MMU
- nested MMU virtualization
- code deprivileging: Nested MMU virtualization works by monitoring all modifications of virtual-to-physical translations (mappings) and explicitly removing MMU modifying instructions from the unprivileged component.

Gatekeeper: inside secmon. Controls all comms:
- VMs -> hypervisor (exits) and
- hypervisor -> VMs (entries)

shared service:
- Scheduler, 
- Memory Management, (= "domain management"?)
- Event Channels 
(all or partly)

per-VM slices:
- Code Emulation
- I/O 
- nested page table management

Limitations:

(1) vulnerabilities in the shared service. In our design, shared service is a unique component and is shared by all Xen slices. If a logic error residing in this part is exploited, the hypervisor may be compromised.
 
(2) abuse of I/O devices. For example, disks are not managed by Nexen, which may be exhausted to cause a DoS.

(3) Nexen is unable to capture all iret instructions used to return to a PV guest currently; a PV guest’s Xen slice compromised by other VMs can bypass the gate keeper’s sanity check and arbitrarily modify the guest’s running state. Fortunately, this can only result from a malicious administrator.


=================
Other projects...

HyperSafe proposes "non-bypassable memory lockdown": 
- gathers all the MMU operations to a specific module 
- deprivileges other modules to do similar operations. 

HyperSafe focuses on protection of hypervisor’s control flow integrity (CFI), Nexen considers Xen decomposition and deprivileging besides CFI. 

Nested Kernel [14] further provides MMU virtualization as a primitive of operating system to enhance the security of all kinds of kernel modules.



https://www.reddit.com/r/rust/comments/52wzl1/redox_is_now_in_the_github_open_source_operating/

seL4 imposes a very strict minimality principle. only implements three services in the kernel: 
- Address space isolation, 
- synchronous IPC with asynchronous notification, and 
- scheduling. 
- Timers are inside the kernel only because scheduling requires them; everything else is in userspace.

Accurate? "A well-designed kernel API/ABI provides isolation that is as good as or better than virtualization, full stop. See seL4."

"seL4 implements the process abstraction somewhere very near "perfectly". (Muen SK, possibly, does the same for virtualization)."

=========
KVM v Xen 

http://www.linux-kvm.org/page/FAQ#What_is_the_difference_between_KVM_and_Xen.3F

- KVM only run on processors that supports x86 hvm (vt/svm instructions set) 
- Xen also allows running modified operating systems on non-hvm x86 processors (via paravirtualization). 

- KVM does not support paravirtualization for CPU 
- KVM may support paravirtualization for device drivers to improve I/O performance.


================
Vbox genode nova

Ported code
400,000 lines of code (sloccount)

New code
6,200 lines (sloccount)
hm, iommio, ioport, mm, pdm, pgm, sup

Modifications of the original code
510 lines added
120 lines removed

-framebuffer VESA
-input ps2
-rom
-filesystem
-block AHCI



==============
random stuff


http://retis.sssup.it/~fabio/soc07/clog.php
http://retis.sssup.it/~fabio/soc07/PROPOSAL


https://www.linux-kvm.org/images/7/71/2011-forum-porting-to-smartos.pdf

It looks like the NOVA VMM might only support 32bit, so no good:
> ports/src/app/seoul
Seoul is a virtual-machine monitor developed for the use with the NOVA platform. It virtualizes 32bit x86 PC hardware including various peripherals.

http://genode.org/documentation/components

Archi of the kvm, linux-kongress.org 2010

Freebsd wiki fabiochecconi porting kvm

Kernelvirtualmachine indian inst of tech bombay

Tldp.org tous of the linux kernel source

Kvm architecture overview google slides

Haifux.org high-level intro to virt's low-level

PSTviewtool.codeplex.com

Epita.fr kvm without qemu

Static analysis of the xen kernel using frama-c , jucs.org


Hypervisors and virtual machines - usenix

Re-engineering xen internals for higher-assurance security

The xen approach, cs.vt.edu

Subverting the xen hypervisor, blackhat

Definitive guide to the xen hypervisor, informIT

Deconstructing xen , NDSS 2017, morning paper

An overview of xen virtualization, dell


http://www.linux-kvm.org/page/Code#building_an_external_module_with_older_kernels

=================
KVMarx setup

Repos to clone:
- KVM kernel
- kvm-kmod
- QEMU, because that is where the KVM userspace code lives

- genode
- redox kernel

- redox newlibc or genode libc?

Our repos:
-kvMar
…vagrant setup

================================
Balasubramanian et al., HotOS’17

-  Software Fault Isolation (SFI) library in Rust, which supports secure communication across isolation boundaries with negligible overhead. The library exports two data types, protection domains (PDs) and remote references (rrefs).
- proof-of-concept Static information flow control (IFC)  for Rust. Rust macros transform a program into an abstract representation in which the value of each variable is simply represented by its security label. 
- automatic checkpointing library for Rust following this observation using a Checkpointable trait (interface) and a custom implementation for Rc (Arc could be extended similarly). A compiler plugin automatically generates an implementation of the Checkpointable trait for types composed of scalar values and references to other checkpointable types.

