
Hey fellow comrade, happy to see you.

Just wanna talk about my experience flashing OpenWRT on a TP Link TD-W8960N V5 and some general investigation on the router original firmware I did.

First thing I did was scanning the router for running services on it, using nmap I got

    Starting Nmap 7.70 ( https://nmap.org ) at 2019-05-27 10:47 CEST
    Nmap scan report for TP-LINK.Home (192.168.1.1)
    Host is up (0.0092s latency).
    Not shown: 998 closed ports
    PORT   STATE SERVICE VERSION
    23/tcp open  telnet  TP-LINK TD-W8960N WAP telnetd
    80/tcp open  http    micro_httpd
    Service Info: Device: WAP; CPE: cpe:/h:tp-link:td-w8960n

    Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
    Nmap done: 1 IP address (1 host up) scanned in 7.63 seconds


I decided to see if I could log into telnet and just using the default credentials admin:admin worked.
On telnet I typed help to see what I could do and I got this output

     > help
    ?
    help
    logout
    exit
    quit
    reboot
    adsl
    xdslctl
    xtm
    brctl
    cat
    loglevel
    logdest
    virtualserver
    ddns
    dnsproxy
    syslog
    echo
    ifconfig
    ping
    ps
    pwd
    sntp
    tftp
    wlctl
    arp
    defaultgateway
    dhcpserver
    dns
    lan
    lanhosts
    passwd
    ppp
    restoredefault
    route
    save
    swversion
    uptime
    cfgupdate
    swupdate
    exitOnIdle
    wan
    mcpctl
    utility


All commands are pretty self explanatory. I focused my attention on the tftp command as I wanted to install custom software on my router and I needed to update such software on the router first

     > tftp
    BusyBox v1.17.2 (2015-12-24 17:10:59 HKT) multi-call binary.

    Usage: tftp [OPTIONS] HOST [PORT]

    Transfer a file from/to tftp server

    Options:
            -l FILE Local FILE
            -r FILE Remote FILE
            -g      Get file
            -p      Put file
            -g -t i -f filename server_ip   Get (flash) broadcom or whole image to modem
            -g -t c -f filename server_ip   Get (flash) config file to modem
            -p -t f -f filename server_ip   Put (backup) config file to tftpd server



This is pretty awesome. I can even flash custom firmware images from this using the -t switch (I guess it stands for type) with the i option (stands for image I guess) and the -f switch (stands for flash probably).

Turns out that on OpenWRT website there is a page about my router model that actually documents this tftp tool on TP Link routers and teaches you how to flash firmware using it. So all the work I did was useless, but it was a lot of fun to find this tool out myself.

Among the commands on the router shell you can see also cat so I decided to cat /etc/passwd to have some password hashes and a user list

    > cat /etc/passwd
    admin:2bfSUcnAaGGQ2:0:0:Administrator:/:/bin/sh
    support:hwhUpqJlIZ5YA:0:0:Technical Support:/:/bin/sh
    user:qoNV962jwQaFA:0:0:Normal User:/:/bin/sh
    nobody:yI3dZH8Ed4WHQ:0:0:nobody for ftp:/:/bin/sh


I had all what I needed, but I was still pretty curious about the router and willing to find some other stuff about it.
I decided to download a firmware update from TP Link website and run binwalk to see how it is formatted

    ┌─[ekardnam@parrot]─[~/tplink/TD-W8960N_V5_141107]
    └──╼ $binwalk TD-W8960Nv5_un_1_1_1_141107R34856.bin 

    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             TP-Link firmware header, firmware version: 0.0.-32550, image version: "", product ID: 0x0, product version: 144048128, kernel load address: 0x0, kernel entry point: 0x80010000, kernel offset: 5968507, kernel length: 0, rootfs offset: 692, rootfs length: 0, bootloader offset: 2854912, bootloader length: 0
    512           0x200           Broadcom 96345 firmware header, header size: 256, firmware version: "8", board id: "6318REF", ~CRC32 header checksum: 0x869C8FCF, ~CRC32 data checksum: 0xAF0F92E9
    768           0x300           Squashfs filesystem, little endian, non-standard signature, version 4.0, compression:gzip, size: 4890804 bytes, 865 inodes, blocksize: 65536 bytes, created: 2014-11-07 01:50:04
    4895500       0x4AB30C        LZMA compressed data, properties: 0x6D, dictionary size: 4194304 bytes, uncompressed size: 3175500 bytes

Using the -e switch I could also extract the squashfs filesystem and look at it. Unlickily I tried updating from the webpage by uploading the firmware image, but router said it was corrupted and could not be installed.

I decided to go for tftp to install the official firmware image. I used [TFTPy](http://tftpy.sourceforge.net) and used a script found in the documentation

    import tftpy

    server = tftpy.TftpServer('.') # use current folder
    server.listen('0.0.0.0', 69)


And this is what I got on the router

     > tftp -g -t i -f TD-W8960Nv5_un_1_1_1_141107R34856.bin 192.168.1.100
    tftp: Allocating 8388628 bytes for flash image.

    tftp: Memory allocated

    tftp: Got image via tftp, total image size: 5969019

    tftp:error:418.511:verifyTplinkFileTag:485:the SW_VER version is bigger than the upgrate version!
    tftp:error:418.512:getVersionLimit:245:the model is W8960N, the versionLimit is 1.1.1
    !
    tftp:error:418.513:verifyTplinkFileTag:491:the Firmware version is too low to upgrate!
    tftp:error:418.513:cmsImg_validateImage:1032:Invalid TPLINK image
    tftp: Tftp Image failed: Illegal image.


At least this time it was telling me what was wrong with the image, apparently the firmware version is too low (?), I thought I got the newer one really.

At this point I decided to download the OpenWRT image to see how it's made. https://openwrt.org/toh/tp-link/td-w8960n said to check this link for V5 of the hardware: https://openwrt.org/toh/plusnet/fast2704nv1 
This is what the OpenWRT image looks like

    ┌─[✗]─[ekardnam@parrot]─[~/tplink/TD-W8960N_V5_141107]
    └──╼ $binwalk openwrt-18.06.2-brcm63xx-smp-FAST2704N-squashfs-cfe.bin 

    DECIMAL       HEXADECIMAL     DESCRIPTION
    --------------------------------------------------------------------------------
    0             0x0             Broadcom 96345 firmware header, header size: 256, firmware version: "8", board id: "@ST2704N", ~CRC32 header checksum: 0xB8ABA22E, ~CRC32 data checksum: 0xB8C8F9E1
    1521100       0x1735CC        Squashfs filesystem, little endian, version 4.0, compression:xz, size: 2468450 bytes, 1171 inodes, blocksize: 262144 bytes, created: 2019-01-30 12:21:


Before doing so I even tried removing the TP Link firmware header stuff from the official TP Link image as the tftp help said `Get (flash) broadcom or whole image to modem` I thought that maybe I could flash it without the TP Link header, but had no luck:

     > tftp -g -t i -f image.bin 192.168.1.100                            
    tftp: Allocating 8388628 bytes for flash image.

    tftp: Memory allocated

    tftp: Got image via tftp, total image size: 5968507

    tftp:error:376.946:verifyTplinkFileTag:428:Image totalImageLen is not correct!
    tftp:error:376.946:cmsImg_validateImage:1032:Invalid TPLINK image
    tftp: Tftp Image failed: Illegal image.


Actually reading more on the OpenWRT wiki and looking on the internet I found a way to flash this image with the stripped TP Link header on the router. If you press the reset button while the router is booting you have a special interface, read at https://openwrt.org/docs/techref/bootloader/cfe
On the web interface I uploaded the TP Link header stripped image and the router was happy to flash it.

At this time I should try using the same technique to flash the OpenWRT firmware on the router.
Oh something I did not tell you is that when booting the router in this mode the wifi stuff is not turned on so you must connect via ethernet and also no DHCP service is running so you have to set an ip manually.

And bang I had OpenWRT flashed and running!

After setting a password and enabling ssh

    ┌─[✗]─[ekardnam@parrot]─[~]
    └──╼ $ssh root@192.168.1.1
    root@192.168.1.1's password: 


    BusyBox v1.28.4 () built-in shell (ash)

      _______                     ________        __
     |       |.-----.-----.-----.|  |  |  |.----.|  |_
     |   -   ||  _  |  -__|     ||  |  |  ||   _||   _|
     |_______||   __|_____|__|__||________||__|  |____|
              |__| W I R E L E S S   F R E E D O M
     -----------------------------------------------------
     OpenWrt 18.06.2, r7676-cddd7b4c77
     -----------------------------------------------------


And done. Happy hacking!





