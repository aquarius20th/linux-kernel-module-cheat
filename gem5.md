# GEM5

## Introduction

GEM5 is a system simulator, much like QEMU: <http://gem5.org/>

Vs QEMU:

-   advantage: simulates a generic more realistic pipelined and optionally out of order CPU cycle by cycle, including a realistic DRAM memory access model with latencies, caches and page table manipulations. This allows us to:

    - do much more realistic performance benchmarking with it, which makes absolutely no sense in QEMU, which is purely functional
    - make functional cache observations, e.g. to use Linux kernel APIs that flush memory like DMA, which are crucial for driver development. In QEMU, the driver would still work even if we forget to flush caches.

    It is not of course truly cycle accurate, as that would require exposing proprietary information of the CPU designs: <https://stackoverflow.com/questions/17454955/can-you-check-performance-of-a-program-running-with-qemu-simulator/33580850#33580850>, but the approximation is reasonable.

    It is used mostly for research purposes: when you are making a new chip technology, you don't really need to specialize enormously to an existing microarchitecture, but rather develop something that will work with a wide range of future architectures.

-   disadvantage: slower than QEMU by TODO 10x?

    This also implies that the user base is much smaller, since no Android devs.

    Instead, we have only chip makers, who keep everything that really works closed, and researchers, who can't version track or document code properly >:-) And this implies that:

    - the documentation is more scarce
    - it takes longer to support new hardware features

## ARM

    ./configure && ./build -a arm-gem5
    ./rungem5 -a arm-gem5

On another shell:

    ./rungem5-shell

### Kernel command line arguments

E.g., to add `printk.time=y`, run:

    ./rungem5 -a arm-gem5 -- --command-line='earlyprintk=pl011,0x1c090000 console=ttyAMA0 lpj=19988480 norandmaps rw loglevel=8 mem=512MB root=/dev/sda printk.time=y'

When you use `--command-line=`, it overrides default command lines, which are required to boot properly.

So if you pass just `--command-line='printk.time=y'`, it removes the required options, and boot fails.

An easy way to find the other options is to to an initial boot:

    ./rungem5 -a arm-gem5

and then look at the line of the linux kernel that starts with

    Kernel command line:

We might copy the default `--command-line` into our startup scripts to make things easier at some point, but it would be fun to debug when the defaults change upstream and we don't notice :-(

### QEMU with GEM5 kernel configuration

TODO: QEMU did not work with the GEM5 kernel configurations.

To test this, hack up `run` to use the `buildroot/output.arm-gem5~` directory, and then run:

    ./run -a arm

Now QEMU hangs at:

    audio: Could not init `oss' audio driver

and the display shows:

    Guest has not initialized the display (yet).

### GEM5 with QEMU kernel configuration

Test it out with:

    ./rungem5 -a arm

TODO hangs at:

    **** REAL SIMULATION ****
    warn: Existing EnergyCtrl, but no enabled DVFSHandler found.
    info: Entering event queue @ 0.  Starting simulation...
    1614868500: system.terminal: attach terminal 0

and the `telnet` at:

    2017-12-28-11-59-51@ciro@ciro-p51$ ./rungem5-shell
    Trying 127.0.0.1...
    Connected to localhost.
    Escape character is '^]'.
    ==== m5 slave terminal: Terminal 0 ====

I have also tried to copy the exact same kernel command line options used by QEMU, but nothing changed.

## x86

TODO didn't get it working yet.

Related threads:

- <https://www.mail-archive.com/gem5-users@gem5.org/msg11384.html>
- <https://stackoverflow.com/questions/37906425/booting-gem5-x86-ubuntu-full-system-simulation>
- <http://www.lowepower.com/jason/creating-disk-images-for-gem5.html> claims to have a working config for x86_64 kernel 4.8.13

### Our best attempt

    ./configure && ./build -a x86_64-gem5
    ./rungem5 -a x86_64-gem5

telnet:

    i8042: PNP: No PS/2 controller found.
    i8042: Probing ports directly.
    Connection closed by foreign host.

stdout:

    panic: Data written for unrecognized command 0xd1
    Memory Usage: 1235908 KBytes
    Program aborted at tick 427627410500

The same failure happens if we use the working QEMU Linux kernel, and / or if we use the kernel 4.8.13 as proposed in lowepower's post..

If we look a bit into the source, the panic message comes from `i8042.cc`, and on the header we see that the missing command is:

        WriteOutputPort = 0xD1,

The kernel was compiled with `CONFIG_SERIO_I8042=y`, I didn't dare disable it yet. The Linux kernel driver has no `grep` hits for either of `0xd1` nor `output.?port`, it must be using some random bitmask to build it then.

This byte is documented at <http://wiki.osdev.org/%228042%22_PS/2_Controller>, as usual :-)

There are also a bunch of `i8042` kernel CLI options, I tweaked all of them but nothing.

### Working baseline with magic image

Working x86 with the pre-built magic image with an ancient 2.6.22.9 kernel starting point:

    sudo mkdir -p /dist/m5/system
    sudo chmod 777 /dist/m5/system
    cd /dist/m5/system
    # Backed up at:
    # https://github.com/cirosantilli/media/releases/tag/gem5
    wget http://www.gem5.org/dist/current/x86/x86-system.tar.bz2
    tar xvf x86-system.tar.bz2
    cd x86-system
    dd if=/dev/zero of=disks/linux-bigswap2.img bs=1024 count=65536
    mkswap disks/linux-bigswap2.img
    cd ..

    git clone https://gem5.googlesource.com/public/gem5
    cd gem5
    git checkout da79d6c6cde0fbe5473ce868c9be4771160a003b
    scons -j$(nproc) build/X86/gem5.opt
    # That old blob has wrong filenames.
    ./build/X86/gem5.opt \
        -d /tmp/output \
        --disk-image=/dist/m5/system/disks/linux-x86.img \
        --kernel=/dist/m5/system/binaries/x86_64-vmlinux-2.6.22.9 \
        configs/example/fs.py

On another shell:

    telnet localhost 3456

### Unmodified Buildroot images 2

bzImage fails, so we always try with vmlinux obtained from inside build/.

rootfs.ext2 and vmlinux from 670366caaded57d318b6dbef34e863e3b30f7f29ails as:

Fails as:

    Global frequency set at 1000000000000 ticks per second
    warn: DRAM device capacity (8192 Mbytes) does not match the address range assigned (512 Mbytes)
    info: kernel located at: /data/git/linux-kernel-module-cheat/buildroot/output.x86_64~/build/linux-custom/vmlinux
    Listening for com_1 connection on port 3456
        0: rtc: Real-time clock set to Sun Jan  1 00:00:00 2012
    0: system.remote_gdb.listener: listening for remote gdb #0 on port 7000
    warn: Reading current count from inactive timer.
    **** REAL SIMULATION ****
    info: Entering event queue @ 0.  Starting simulation...
    warn: instruction 'fninit' unimplemented
    warn: Don't know what interrupt to clear for console.
    12516923000: system.pc.com_1.terminal: attach terminal 0
    warn: i8042 "Write output port" command not implemented.
    warn: i8042 "Write keyboard output buffer" command not implemented.
    warn: Write to unknown i8042 (keyboard controller) command port.
    hack: Assuming logical destinations are 1 << id.
    panic: Resetting mouse wrap mode unimplemented.
    Memory Usage: 1003456 KBytes
    Program aborted at tick 632745027500
    --- BEGIN LIBC BACKTRACE ---
    ./build/X86/gem5.opt(_Z15print_backtracev+0x15)[0x12b8165]
    ./build/X86/gem5.opt(_Z12abortHandleri+0x39)[0x12c32f9]
    /lib/x86_64-linux-gnu/libpthread.so.0(+0x11390)[0x7fe047a71390]
    /lib/x86_64-linux-gnu/libc.so.6(gsignal+0x38)[0x7fe046601428]
    /lib/x86_64-linux-gnu/libc.so.6(abort+0x16a)[0x7fe04660302a]
    ./build/X86/gem5.opt(_ZN6X86ISA8PS2Mouse11processDataEh+0xf5)[0x1391095]
    ./build/X86/gem5.opt(_ZN6X86ISA5I80425writeEP6Packet+0x51c)[0x13927ec]
    ./build/X86/gem5.opt(_ZN7PioPort10recvAtomicEP6Packet+0x66)[0x139f7b6]
    ./build/X86/gem5.opt(_ZN15NoncoherentXBar10recvAtomicEP6Packets+0x200)[0x1434af0]
    ./build/X86/gem5.opt(_ZN6Bridge15BridgeSlavePort10recvAtomicEP6Packet+0x5d)[0x140ee9d]
    ./build/X86/gem5.opt(_ZN12CoherentXBar10recvAtomicEP6Packets+0x3e7)[0x1415b77]
    ./build/X86/gem5.opt(_ZN15AtomicSimpleCPU8writeMemEPhjm5FlagsIjEPm+0x327)[0xa790a7]
    ./build/X86/gem5.opt(_ZN17SimpleExecContext8writeMemEPhjm5FlagsIjEPm+0x19)[0xa856b9]
    ./build/X86/gem5.opt(_ZNK10X86ISAInst2St7executeEP11ExecContextPN5Trace10InstRecordE+0x235)[0xfb9e65]
    ./build/X86/gem5.opt(_ZN15AtomicSimpleCPU4tickEv+0x23c)[0xa784fc]
    ./build/X86/gem5.opt(_ZN10EventQueue10serviceOneEv+0xc5)[0x12be0d5]
    ./build/X86/gem5.opt(_Z9doSimLoopP10EventQueue+0x38)[0x12cd558]
    ./build/X86/gem5.opt(_Z8simulatem+0x2eb)[0x12cdbdb]
    ./build/X86/gem5.opt(_ZZN8pybind1112cpp_function10initializeIRPFP22GlobalSimLoopExitEventmES3_ImEINS_4nameENS_5scopeENS_7siblingENS_5arg_vEEEEvOT_PFT0_DpT1_EDpRKT2_ENUlRNS_6detail13function_callEE1_4_FUNESO_+0x41)[0x13fca11]
    ./build/X86/gem5.opt(_ZN8pybind1112cpp_function10dispatcherEP7_objectS2_S2_+0x8d8)[0xfc7398]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x7852)[0x7fe047d3b552]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fe047e6501c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x6ffd)[0x7fe047d3acfd]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x7124)[0x7fe047d3ae24]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x7124)[0x7fe047d3ae24]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fe047e6501c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCode+0x19)[0x7fe047d33b89]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x613b)[0x7fe047d39e3b]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fe047e6501c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x6ffd)[0x7fe047d3acfd]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fe047e6501c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCode+0x19)[0x7fe047d33b89]
    --- END LIBC BACKTRACE ---
    Aborted (core dumped)

Boot goes quite far, on telnet:

    ALSA device list:
      No soundcards found.

So just looks like we have to disable some Linux configs which GEM5 does not support... so fragile.

### Copy upstream 2.6 configs on 4.9 kernel

The magic image provides its kernel configurations, so let's try that.

The configs are present at:

    wget http://www.gem5.org/dist/current/x86/config-x86.tar.bz2

backed up at: <https://github.com/cirosantilli/media/releases/tag/gem5>

Copy `linux-2.6.22.9` into the kernel tree as `.config`, `git checkout v4.9.6`, `make olddefconfig`, `make`, then use the Buildroot filesystem as above, failure:

    panic: Invalid IDE control register offset: 0
    Memory Usage: 931272 KBytes
    Program aborted at tick 382834812000
    --- BEGIN LIBC BACKTRACE ---
    ./build/X86/gem5.opt(_Z15print_backtracev+0x15)[0x12b8165]
    ./build/X86/gem5.opt(_Z12abortHandleri+0x39)[0x12c32f9]
    /lib/x86_64-linux-gnu/libpthread.so.0(+0x11390)[0x7fc2081c6390]
    /lib/x86_64-linux-gnu/libc.so.6(gsignal+0x38)[0x7fc206d56428]
    /lib/x86_64-linux-gnu/libc.so.6(abort+0x16a)[0x7fc206d5802a]
    ./build/X86/gem5.opt(_ZN7IdeDisk11readControlEmiPh+0xd9)[0xa96989]
    ./build/X86/gem5.opt(_ZN13IdeController14dispatchAccessEP6Packetb+0x53e)[0xa947ae]
    ./build/X86/gem5.opt(_ZN13IdeController4readEP6Packet+0xe)[0xa94a5e]
    ./build/X86/gem5.opt(_ZN7PioPort10recvAtomicEP6Packet+0x3f)[0x139f78f]
    ./build/X86/gem5.opt(_ZN15NoncoherentXBar10recvAtomicEP6Packets+0x200)[0x1434af0]
    ./build/X86/gem5.opt(_ZN6Bridge15BridgeSlavePort10recvAtomicEP6Packet+0x5d)[0x140ee9d]
    ./build/X86/gem5.opt(_ZN12CoherentXBar10recvAtomicEP6Packets+0x3e7)[0x1415b77]
    ./build/X86/gem5.opt(_ZN15AtomicSimpleCPU7readMemEmPhj5FlagsIjE+0x3ef)[0xa780ef]
    ./build/X86/gem5.opt(_ZN17SimpleExecContext7readMemEmPhj5FlagsIjE+0x11)[0xa85671]
    ./build/X86/gem5.opt(_ZNK10X86ISAInst2Ld7executeEP11ExecContextPN5Trace10InstRecordE+0x130)[0xfb6c00]
    ./build/X86/gem5.opt(_ZN15AtomicSimpleCPU4tickEv+0x23c)[0xa784fc]
    ./build/X86/gem5.opt(_ZN10EventQueue10serviceOneEv+0xc5)[0x12be0d5]
    ./build/X86/gem5.opt(_Z9doSimLoopP10EventQueue+0x38)[0x12cd558]
    ./build/X86/gem5.opt(_Z8simulatem+0x2eb)[0x12cdbdb]
    ./build/X86/gem5.opt(_ZZN8pybind1112cpp_function10initializeIRPFP22GlobalSimLoopExitEventmES3_ImEINS_4nameENS_5scopeENS_7siblingENS_5arg_vEEEEvOT_PFT0_DpT1_EDpRKT2_ENUlRNS_6detail13function_callEE1_4_FUNESO_+0x41)[0x13fca11]
    ./build/X86/gem5.opt(_ZN8pybind1112cpp_function10dispatcherEP7_objectS2_S2_+0x8d8)[0xfc7398]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x7852)[0x7fc208490552]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fc2085ba01c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x6ffd)[0x7fc20848fcfd]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x7124)[0x7fc20848fe24]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x7124)[0x7fc20848fe24]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fc2085ba01c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCode+0x19)[0x7fc208488b89]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x613b)[0x7fc20848ee3b]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fc2085ba01c]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalFrameEx+0x6ffd)[0x7fc20848fcfd]
    /usr/lib/x86_64-linux-gnu/libpython2.7.so.1.0(PyEval_EvalCodeEx+0x85c)[0x7fc2085ba01c]
    --- END LIBC BACKTRACE ---
    Aborted (core dumped)

### Use upstream 2.6 configs and 2.6 kernel

If we checkout to the ancient kernel `v2.6.22.9`, it fails to compile with modern GNU make 4.1: <https://stackoverflow.com/questions/35002691/makefile-make-clean-why-getting-mixed-implicit-and-normal-rules-deprecated-s> lol
