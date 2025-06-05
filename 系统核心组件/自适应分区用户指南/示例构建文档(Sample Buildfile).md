# 示例构建文档(Sample Buildfile)

以下构建文档展示了如何添加自适应分区模块到**procnto**。它也包含了一些创建分区、在分区启动程序与设置安全等级等相关命令。

Note:

​	在真实的构建文档中，你不等你使用反斜杠(\\)来分割一个较长的行为较短的分段，此处文档中这么做，仅仅是为了使构建文档易读。

```sh
[compress=3]
[virtual=x86,bios] .bootstrap = {
    startup-x86

    # PATH is the *safe* path for executables (confstr(_CS_PATH...))
    # LD_LIBRARY_PATH is the *safe* path for libraries
    # (confstr(_CS_LIBPATH)). That is, it's the path searched for libs
    # in setuid/setgid executables.

    # The module=aps enables the adaptive partitioning scheduler.

    [module=aps] PATH=/proc/boot:/bin:/usr/bin:/sbin:/usr/sbin \
        LD_LIBRARY_PATH=/proc/boot:/lib:/lib/dll:/usr/lib \
        procnto-smp-instr 
}

# Start-up script

[+script] .script = {
    # Programs require the runtime linker (ldqnx.so) to be at a fixed
    # location. To save memory, make everyone use the libc in the boot
    # image. For speed (fewer symbolic lookups), we point to libc.so.4
    # instead of libc.so.
    procmgr_symlink ../../proc/boot/libc.so.4 /usr/lib/ldqnx.so.2

    # Create some adaptive partitions during system startup:
    #   - IOPKT with a 20% budget
    #   - QCONN with a 20% budget
    # NOTE: To specify a critical budget of 5 ms, use sched_aps as seen below
    #       when the filesystem on the disk is available.
    sched_aps IOPKT 20 5
    sched_aps QCONN 20 5

    # Start the system logger.
    slogger2 &
    dumper

    # Start the PCI server.
    pci-server &
    waitfor /dev/pci

    display_msg .
    display_msg Welcome to QNX Neutrino 7.0!
    display_msg BIOS system with APS enabled

    # Get the disk up and running.
    devb-eide blk automount=hd0t179:/ &

    waitfor /bin

    # Further commands can now be run from disk.

    # USB services
    display_msg "Start USB services..."
    io-usb-otg -dehci &
    waitfor /dev/usb/io-usb-otg 4
    waitfor /dev/usb/devu-hcd-ehci.so 4

    display_msg "Starting Input services..."
    io-hid -d ps2ser kbd:kbddev:ps2mouse:mousedev \
       -d usb /dev/usb/io-usb-otg &
    waitfor /dev/io-hid/io-hid 10

    # Start up some consoles.
    display_msg Starting consoles
    devc-pty &
    devc-con-hid -n4 &
    reopen /dev/con1

    display_msg Starting serial port driver
    devc-ser8250 -b115200 &

    # Start the networking manager in the IOPKT partition.
    on -X aps=IOPKT io-pkt-v4-hc -d /lib/dll/devnp-abc100.so

    # Start some services.
    pipe
    random -t

    if_up -p wm0
    dhclient -m wm0

    #waitfor /dev/dbgmem

    # Create an additional parition with services:
    aps create -b10 INETD
    on -X aps=INETD inetd &
    on -X aps=QCONN qconn &

    # Specify a critical budget for a partition created during startup:
    aps modify -B 10 IOPKT

    # Use the "recommended" security level for the partitions:
    aps modify -s recommended

    # These env variables are inherited by all the programs that follow:
    SYSNAME=nto
    TERM=qansi

    # Start some extra shells on other consoles:
    reopen /dev/con2
    [+session] sh &
    reopen /dev/con3
    [+session] sh &
    reopen /dev/con4
    [+session] sh &

    # Start the main shell
    reopen /dev/con1
    [+session] sh &
}

[perms=0777] 
# Include the current "libc.so". It will be created as a real file
# using its internal "SONAME", with "libc.so" being a symlink to it.
# The symlink will point to the last "libc.so.*" so if an earlier
# libc is needed (e.g. libc.so.4) add it before the this line.
libc.so
libelfcore.so.1
libslog2.so


#######################################################################
## uncomment for USB driver
#######################################################################
#libusbdi.so
devu-hcd-ehci.so

fs-qnx6.so
cam-disk.so
io-blk.so
libcam.so

devnp-abc100.so
#libsocket.so

devc-con
pci-server
devc-ser8250
dumper
devb-eide
io-pkt-v4-hc
mount
slogger2
sh
cat
ls
pidin
less
on
qconn
```



