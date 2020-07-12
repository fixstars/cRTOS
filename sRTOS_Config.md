sRTOS Nuttx Configuration details
=================================


CONFIG_CRTOS
------------
This setting enables the cRTOS functionality in Nuttx.

CONFIG_TUX_FD_RESERVE
---------------------
Number of file descriptors reserved for Linux files in Nuttx.

The multiplexing of file in Linux and Nuttx are done via segregation of file descriptors.
Number lower than this are Linux files, higher than this are Nuttx files.

This setting will limit number of Linux files you can access in a process.


CONFIG_TUX_USER_ADDR_START
--------------------------

This is the starting physical address of the memory pool for user pages for processes.

CONFIG_TUX_USER_ADDR_SIZE
-------------------------

This is the size of the memory pool for user pages for processes.


CONFIG_VIRT_SHADOW
------------------

This enables the shadow device driver for remote system call.

CONFIG_VIRT_SHADOW_BASE_IRQ
---------------------------

This enables the IRQ number used for shadow device driver.

CONFIG_SYSTEM_CRTOS_DAEMON
--------------------------
This enables the program loader daemon for loading program.

The following setting are used for configuring the network interface which the cRTOS program loading daemon listens to.
 * CONFIG_SYSTEM_CRTOS_DAEMON_NIC="eth0"
 * CONFIG_SYSTEM_CRTOS_DAEMON_IPADDR=0xac100002
 * CONFIG_SYSTEM_CRTOS_DAEMON_NETMASK=0xffffff00
 * CONFIG_SYSTEM_CRTOS_DAEMON_DRIPADDR=0xac100001

The following setting of TCP setting of cRTOS daemon.
 * CONFIG_SYSTEM_CRTOS_DAEMON_PORT=42
 * CONFIG_SYSTEM_CRTOS_DAEMON_BACKLOG=8
