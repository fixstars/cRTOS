Compounded Real-time Operating System (cRTOS)
=============================================

Introduction
------------

Compounded Real-time operating system (cRTOS) is an RTOS providing real-time capabilities of swift RTOS and richfulness of standard GPOS.
This is accomplished by running a Linux (GPOS) and a Nuttx (sRTOS) in parallel on a single machine by Jailhouse (hypervisor).

The process is executed inside Nuttx and time-critical system calls are handled locally in Nuttx.
While complicated system calls, credentials, etc, are  handled on Linux side.

We allow standard Linux executable ELF to reused without recompilation.
Most of the standard Linux programs will work out of the box (including those you can get from apt, yum, pacman).
In addition, you can write programs as a normal Linux programmer, more cumbersome RTOS APIs!

For device drivers, by exploiting the everything is file concept of common *nix, access to devices/files/system services are multiplexed by the location of the corresponding file.
The root filesystem are overlayed as Nuttx's shadowing Linux's. 
The system calls will first try on Nuttx files before Linux files.
You can not only access a real-time device, enumerated in Nuttx `/dev` with Nuttx driver, but also non-real-time devices enumerated in Linux `/dev` with Linux driver.
You can access a named pipe in Linux filesystem to talk to other Linux processes, or a named pipe in Nuttx filesystem to talk to other processes.
This provides the flexibility on creating a real-time system, not to port write drivers for every devices for a RTOS.
You can reuse Linux drivers not only for non-real-time devices, but also real-tiem devices for fast prototyping.

Requirements
------------

### Software
 cRTOS is tested with the packages below.

 * glibc 2
 * Linux kernel 5.4 with PREEMPT_RT
 * GCC 9

### Hardware

cRTOS is utilizing the Jailhouse hypervisor and Nuttx x86_64 port.
The processor should meet the requirement of these two packages.
This includes:
 * A x86_64 processor with VT-x, VT-d, X2APIC and TSC_DEADLINE support
 * At lease one of the traditional AT 8250 serial port (for Jailhouse debug output)
 * At lease one of the MCS99xx PCI-e to serial adapter (for sRTOS debug output)

The development is conducted on a Broadwell machine with the following components.
This is also the only tested machine configuration, but theoretically anything similar should work.
 * CPU: Xeon E5-2650 v4
 * Motherboard: Supermicro C7X99-OCE-F
 * RAM: 16 GB
 * PCI-e serial adaptor 1: MTG-PCIE-EMT02A-1 (MosChip Semiconductor Technology Ltd. PCIe 9912 Multi-I/O Controller)
 * PCI-e serial adaptor 2: MTG-PCIE-EMT02A-1 (MosChip Semiconductor Technology Ltd. PCIe 9912 Multi-I/O Controller)

Installation
------------

Too long, reference the Installation.md for detailed instructions.

Using and Testing
-----------------

Assume have followed Installation.md and setup you system with everything compiled, you can now try to run some programs using the loader.

As a remark, the `loader` have to be executed as either SCHED_FIFO or SCHED_RR.
Anything else then that will not be accepted by loader.
As a result, you must run loader as root user.
Thi can be done easiry using the combination of `sudo` and `chrt`.

First we can try some simple applications, for example `/bin/echo`.
```sh
cd crtos-loader

chrt -f 90 ./loader /bin/echo "Hello World from sRTOS!"
Hello World from sRTOS!
```

`echo` is loader into sRTOS and executed as expected.
Because stdin/stdout/stderr are redirected to the Linux side, it seems like that `echo` is executed locally.

Now, we can do some benchmarks, using the infamous cyclictest with 1ms interval and 1000 sample.
```
sudo apt-get install rt-tests

# This runs it locally in Linux
cyclictest -nq -p 90 -h 200 -l 1000 -i 1000

# This runes in sRTOS, Nuttx
chrt -f 90 ./loader cyclictest -nq -p 90 -h 200 -l 1000 -i 1000
```

A even more complex example, we will run a `/bin/sh` in sRTOS.
```
# This spawn a new shell, backed by sRTOS
chrt -f 90 ./loader /bin/shell

# Any following command are executed in sRTOS and file access are from the sRTOS perspective of view.
```

Publication and Acknowledgement
-------------------------------

This work is published on:
 * SIGPLAN/SIGOPS International Conference on Virtual Execution Environments March 2020 

The paper is on open access:
 * https://doi.org/10.1145/3381052.3381323

Please this work cite as:
> Chung-Fan Yang and Yasushi Shinjo. 2020. Obtaining hard real-time performance and rich Linux features in a compounded real-time operating system by a partitioning hypervisor. In Proceedings of the 16th ACM SIGPLAN/SIGOPS International Conference on Virtual Execution Environments (VEE ’20). Association for Computing Machinery, New York, NY, USA, 5972. DOI:https://doi.org/10.1145/3381052.3381323

In addition, this work is brought to you as an open-source software by:
 * Fixstars corporation. https://www.fixstars.com/
