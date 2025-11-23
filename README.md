# RH25 v2 / PFalcon 640+ v2 reverse engineering notes

## Overview 

The RH25 v2 / PFalcon 640+ v2 is a fairly neat thermal optic distributed by Infiray / iRay.

See https://irayusa.com/infiray-outdoor-rico-micro-v2-640-1x-25mm-multifunction-thermal-weapon-sight/ for more details.

## Executive Summary

The RH25 v2 exposes several unsecured services on its WiFi interface, including a root shell.

This access could be used to completely understand, document, and modify the functionality of the device.
A fully custom firmware is not out of the question, although I have no intentions to develop one.

Using a weak WiFi password or allowing an untrusted user to connect to your device could result in
total compromise of the system, rendering it inoperable or backdoored. This is probably not a substantial
threat for most people, but risk sensitive users should leave wireless communications disabled.

## Connecting

Let's connect to the optic's Wifi SSID from a computer using the configured password.

Default is "RH25 XXXX" and password of "12345678".

We observe that DHCP is configured and our client is issued an IP in 192.168.11.0/24,
with a gateway address of 192.168.11.123.

## Services

If we `nmap` scan the gateway address, we get a few interesting ports.

```
Starting Nmap 7.98 ( https://nmap.org ) at 2025-11-23 02:33 +0000
Nmap scan report for 192.168.11.123
Host is up (0.011s latency).
Not shown: 997 closed tcp ports (conn-refused)
PORT   STATE SERVICE
21/tcp open  ftp
23/tcp open  telnet
80/tcp open  http

Nmap done: 1 IP address (1 host up) scanned in 6.05 seconds
```

Let's try `telnet`:

```
$ telnet 192.168.11.123 23
Trying 192.168.11.123...
Connected to 192.168.11.123.
Escape character is '^]'.

(none) login: root
login: /etc/group: bad record
numid=22,iface=MIXER,name='ADC Mux'
  ; type=ENUMERATED,access=rw------,values=1,items=4
  ; Item #0 'AIL/L'
  ; Item #1 'AIR/R'
  ; Item #2 'AIL/R'
  ; Item #3 'AIR/L'
  : values=0
amixer command executed successfully
numid=15,iface=MIXER,name='Mic2 Bias Switch'
  ; type=BOOLEAN,access=rw------,values=1
  : values=on
amixer command executed successfully
numid=5,iface=MIXER,name='Mic2 Volume'
  ; type=INTEGER,access=rw---R--,values=1,min=0,max=5,step=0
  : values=5
  | dBscale-min=0.00dB,step=4.00dB,mute=0
amixer command executed successfully
numid=3,iface=MIXER,name='Digital Capture Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=23,step=0
  : values=22,22
  | dBscale-min=0.00dB,step=1.00dB,mute=0
amixer command executed successfully
numid=14,iface=MIXER,name='Mic1 Bias Switch'
  ; type=BOOLEAN,access=rw------,values=1
  : values=on
amixer command executed successfully
numid=4,iface=MIXER,name='Mic1 Volume'
  ; type=INTEGER,access=rw---R--,values=1,min=0,max=5,step=0
  : values=3
  | dBscale-min=0.00dB,step=4.00dB,mute=0
amixer command executed successfully
numid=3,iface=MIXER,name='Digital Capture Volume'
  ; type=INTEGER,access=rw---R--,values=2,min=0,max=23,step=0
  : values=23,23
  | dBscale-min=0.00dB,step=1.00dB,mute=0
amixer command executed successfully
# range [0,31]
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <GPU2D> ONERROR: status=-3(gcvSTATUS_OUT_OF_MEMORY) @ alloc_video_buffer(180)
ERROR: <V4L2> set v4l2_format failed!: Device or resource busy
ERROR: <V4L2> v4l2__set_buf:742 v4l2_set_format(ctx, buf_type, v4l2_fmt, width, height) != 0
ERROR: <V4L2> error @ v4l2__set_buf(830)
ERROR: <V4L2> v4l2 alloc buf failed!
H264_encoder_init delay :0 s
whoami
root
[1]+  Segmentation fault         test_recoder
# uname -a
Linux (none) 3.10.14 #1 PREEMPT Tue Feb 18 22:27:50 PST 2025 mips GNU/Linux
```

So a root login works, it's Linux, and it spams us with some error messages including a
backgrounded task of `test_recoder` encountering a segfault. Lovely.

Let's check `/etc/passwd`:

```
# cat /etc/passwd
root:x:0:0:root:/root:/bin/sh
daemon:x:1:1:daemon:/usr/sbin:/bin/sh
bin:x:2:2:bin:/bin:/bin/sh
sys:x:3:3:sys:/dev:/bin/sh
sync:x:4:100:sync:/bin:/bin/sync
mail:x:8:8:mail:/var/spool/mail:/bin/sh
proxy:x:13:13:proxy:/bin:/bin/sh
www-data:x:33:33:www-data:/var/www:/bin/sh
backup:x:34:34:backup:/var/backups:/bin/sh
operator:x:37:37:Operator:/var:/bin/sh
haldaemon:x:68:68:hald:/:/bin/sh
dbus:x:81:81:dbus:/var/run/dbus:/bin/sh
ftp:x:83:83:ftp:/home/ftp:/bin/sh
nobody:x:99:99:nobody:/home:/bin/sh
sshd:x:103:99:Operator:/var:/bin/sh
default:x:1000:1000:Default non-root user:/home/default:/bin/sh
```

A non-root user called `default` exists. We can telnet in as it, but its home directory doesn't
exist so there's not much special there.

What's running? Let's `ps` and `netstat`:

```
# ps auxww
PID   USER     TIME   COMMAND
    1 root       0:03 init
    2 root       0:00 [kthreadd]
    3 root       0:00 [ksoftirqd/0]
    5 root       0:00 [kworker/0:0H]
    6 root       0:00 [kworker/u2:0]
    7 root       0:03 [rcu_preempt]
    8 root       0:00 [rcu_bh]
    9 root       0:00 [rcu_sched]
   10 root       0:00 [khelper]
   11 root       0:00 [kdevtmpfs]
   12 root       0:00 [netns]
   13 root       0:00 [writeback]
   14 root       0:00 [bioset]
   15 root       0:00 [kblockd]
   16 root       0:00 [khubd]
   17 root       0:00 [kworker/0:1]
   18 root       0:00 [cfg80211]
   19 root       0:00 [sw_core]
   20 root       0:00 [kswapd0]
   21 root       0:00 [fsnotify_mark]
   22 root       0:00 [crypto]
   37 root       0:00 [galcore workque]
   38 root       0:03 [galcore daemon ]
   42 root       0:00 [binder]
   43 root       0:00 [dwc2]
   44 root       0:00 [irq/9-jz-asoc-a]
   45 root       0:00 [mmcqd/0]
   46 root       0:00 [krfcommd]
   47 root       0:00 [krdsd]
   48 root       0:00 [deferwq]
   49 root       0:00 [kworker/u2:2]
   50 root       0:00 [wl_event_handle]
   51 root       0:00 [dhd_watchdog_th]
   52 root       0:02 [dhd_dpc]
   53 root       0:00 [dhd_rxf]
   54 root       0:00 [f_mtp]
   55 root       0:00 [file-storage]
   81 root       0:00 [kworker/0:1H]
   83 root       0:00 [jbd2/mmcblk0p8-]
   84 root       0:00 [ext4-dio-unwrit]
  102 root       0:14 /sbin/syslogd -n
  104 root       0:18 /sbin/klogd -n
  117 root       0:00 [jbd2/mmcblk0p5-]
  118 root       0:00 [ext4-dio-unwrit]
  119 root       0:07 ble
  120 root       0:59 superui
  124 root       0:00 [ext4-dio-unwrit]
  140 root       0:00 [ext4-dio-unwrit]
  147 root       0:00 /usr/bin/debuggerd
  162 root       0:00 brcm_patchram_plus --enable_hci --no2bytes --tosleep 200000 --use_baudrate_for_download --baudrate 300000 --pa
  165 root       0:00 busybox tcpsvd -vE 0.0.0.0 21 ftpd -w /storage
  185 root       0:00 httpd -h /var/www/ -c /etc/httpd.conf
  211 root       0:00 /usr/sbin/telnetd -F
  216 root       0:00 {adbserver.sh} /bin/sh /sbin/adbserver.sh 310
  220 root       0:00 /sbin/adbd
  231 root       0:00 servicemanager
  232 root       0:01 mediaserver
  233 root       0:00 stateMachineServer
  234 root       0:00 mtpserver
  235 root       0:00 -/bin/sh
  257 root       2:45 test_recoder
  270 root       0:00 hostapd /tmp/hostapd.conf
  272 root       0:00 udhcpd -fS /etc/udhcpd.conf
  273 root       0:00 [kworker/0:2]
  395 root       0:00 [kworker/u3:0]
  396 root       0:00 [hci0]
  397 root       0:00 [hci0]
  400 root       0:00 [kworker/u3:1]
16728 root       0:00 -sh
18801 root       0:00 gatttool -i hci0 -b 02:80:E1:00:00:AA --char-write-req -a 0x000F -n 0100 --listen
18804 root       0:00 ps auxww
18808 root       0:00 sh -c pidof gatttool > /tmp/gatttoolpid

# netstat -tulpn
Active Internet connections (only servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name
tcp        0      0 0.0.0.0:20201           0.0.0.0:*               LISTEN      120/superui
tcp        0      0 0.0.0.0:80              0.0.0.0:*               LISTEN      185/httpd
tcp        0      0 0.0.0.0:21              0.0.0.0:*               LISTEN      165/busybox
tcp        0      0 0.0.0.0:23              0.0.0.0:*               LISTEN      211/telnetd
tcp        0      0 0.0.0.0:16385           0.0.0.0:*               LISTEN      -
netstat: /proc/net/tcp6: No such file or directory
udp        0      0 0.0.0.0:67              0.0.0.0:*                           272/udhcpd
udp        0      0 0.0.0.0:20200           0.0.0.0:*                           120/superui
netstat: /proc/net/udp6: No such file or directory
```

Cool. Just from a high level, we see:

* `httpd` on port 80 configured from /etc/httpd.conf
* `ftpd` on port 21 serving /storage
* `telnetd` on port 23 (obviously)
* `udhcpd` server
* `gatttool` handling some BLE stuff for the remote, presumably
* `mtpserver` handling file transfers over USB

And some more interesting non-standard stuff:

* SuperUI (?) on port 20201 TCP / 20200 UDP
* Something unknown on port 16385 TCP
* `brcm_patchram_plus` and `test_recoder` doing... something

### HTTP server

Let's look at /etc/httpd.conf:

```
# cat /etc/httpd.conf
P:/api0/productVersion:http://127.0.0.1:80/cgi-bin/cgijson.cgi

P:/api/:http://127.0.0.1:80/cgi-bin/
```

I'm guessing those are in `/var/www`, so let's take a look...

```
# ls -laR /var/www
/var/www:
total 21
drwxrwxr-x    3 root     root          1024 Nov 16  2024 .
drwxrwxr-x    4 root     root          1024 Feb 19  2025 ..
-rwxrwxr-x    1 root     root           298 Nov 16  2024 404.html
-rwxrwxr-x    1 root     root           864 Nov 16  2024 art.html
drwxrwxr-x    2 root     root          1024 Nov 16  2024 cgi-bin
-rwxrwxr-x    1 root     root           595 Nov 16  2024 fail.html
-rwxrwxr-x    1 root     root          1690 Nov 16  2024 flashing.html
-rwxrwxr-x    1 root     root          1368 Nov 16  2024 halfpga.html
-rwxrwxr-x    1 root     root           738 Nov 16  2024 halmic.html
-rwxrwxr-x    1 root     root          4132 Nov 16  2024 mac.html
-rwxrwxr-x    1 root     root           814 Nov 16  2024 style.css
-rwxrwxr-x    1 root     root          1205 Nov 16  2024 update.asp
-rwxrwxr-x    1 root     root          1220 Nov 16  2024 update.html

/var/www/cgi-bin:
total 1076
drwxrwxr-x    2 root     root          1024 Nov 16  2024 .
drwxrwxr-x    3 root     root          1024 Nov 16  2024 ..
-rwxrwxr-x    1 root     root         77600 Feb 19  2025 cgictest.cgi
-rwxrwxr-x    1 root     root         77400 Feb 19  2025 cgijson.cgi
-rwxrwxr-x    1 root     root         82544 Feb 19  2025 getpasswd
-rwxrwxr-x    1 root     root         82624 Feb 19  2025 getssid
-rwxrwxr-x    1 root     root         82888 Feb 19  2025 productInfo2
-rwxrwxr-x    1 root     root         82600 Feb 19  2025 productVersion
-rwxrwxr-x    1 root     root         81540 Feb 19  2025 pushfile.cgi
-rwxrwxr-x    1 root     root        105408 Feb 19  2025 pushfpga.cgi
-rwxrwxr-x    1 root     root         76824 Feb 19  2025 setdevicecfg.cgi
-rwxrwxr-x    1 root     root         84424 Feb 19  2025 setpasswd
-rwxrwxr-x    1 root     root         84452 Feb 19  2025 setssid
-rwxrwxr-x    1 root     root         81836 Feb 19  2025 trio_tmr
-rwxrwxr-x    1 root     root         81544 Feb 19  2025 version
```

From our client (not the telnet session), we can curl the optic and access the static content
without issue:

```
> curl -v 192.168.11.123/halfpga.html
*   Trying 192.168.11.123:80...
* Connected to 192.168.11.123 (192.168.11.123) port 80
* using HTTP/1.x
> GET /halfpga.html HTTP/1.1
> Host: 192.168.11.123
> User-Agent: curl/8.14.1
> Accept: */*
>
* Request completely sent off
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
< Content-type: text/html
< Date: Sat, 22 Nov 2025 18:44:05 GMT
< Connection: close
< Accept-Ranges: bytes
< Last-Modified: Sat, 16 Nov 2024 02:34:04 GMT
< Content-length: 1368
<
<!--
 * @Author: your name
 * @Date: 2021-07-07 18:04:31
 * @LastEditTime: 2021-07-27 16:44:54
 * @LastEditors: Please set LastEditors
 * @Description: In User Settings Edit
 * @FilePath: /infiray_manhattan-ota0707/iot/external/cgic204/webfile/uboot.html
-->
<!DOCTYPE HTML>
<html>
        <head>
                <meta charset="utf-8">
                <title>Uart2serial</title>
                <link rel="stylesheet" href="style.css">
        </head>
        <body>
                <div id="m">
                        <h1>Uart2Serial</h1>
                        <p>You are going to update <strong>U-Boot bootloader</strong> on the device.<br>Please, choose file from your local hard drive and click <strong>Update U-Boot</strong> button.</p>
                        <form action="cgi-bin/pushfpga.cgi" enctype="multipart/form-data" method="post" >
                        指令：<input type="text" name="FPGAInstruct"size="35"><br />
                        返回字符长度<input type="text" name="backjsonsize"size="10" value="9"><br />
                        是否获取返回指令：<input type="text" name="backjson"size="10" value="1"><br />
                        <br><br>
                        <input type="submit" value="Send2Fpga">
                        </form>
                        <div class="i w">
                                <strong>uart2serial Tips</strong>
                                <ul>
                                        <li>shutter:0xf0 0x01 0x02 0x80 0x00 0x80 0xff</li>
                                        <li>colormode :0xf0 0x01 0x02 0x80 0x30 0xb0 0xff</li>
                                        <li>cold:0xf0 0x01 0x02 0x80 0xb0 0x30 0xff</li>
                                        <li>PN read：0xf0 0x01 0x02 0x80 0x92 0x12 0xff</li>
                                </ul>
                        </div>
                </div>
        </body>
</html>
* shutting down connection #
```

Looks like some interesting debug tools for updating firmware, MAC address, etc.

If we try to hit the `/api/` endpoints, `curl` freaks out:

```
> curl -v 192.168.11.123/api0/productVersion
*   Trying 192.168.11.123:80...
* Connected to 192.168.11.123 (192.168.11.123) port 80
* using HTTP/1.x
> GET /api0/productVersion HTTP/1.1
> Host: 192.168.11.123
> User-Agent: curl/8.14.1
> Accept: */*
>
* Request completely sent off
* HTTP 1.0, assume close after body
< HTTP/1.0 200 OK
* Header without colon
* closing connection #0
curl: (8) Header without colon
```

Ok, non-standard HTTP response? Let's craft one manually and check it out:

```
> printf "GET /api0/productVersion HTTP/1.1\r\n\r\n" | nc 192.168.11.123 80
HTTP/1.0 200 OK
cgijson start ------------>
Content-type: application/json

{ "code": 0, "bootloader": "200908", "gui": "0123", "fpga": "FPGA:1319,PN:M2-312,SN:2170XXXXXX", "firmware": "202103121535_common", "fwid": 12288 }
```

Yup, they put some weird debug output in the middle of the headers. Oh well.

Let's quickly check the other endpoints:

* `/api/getssid` - returns the configured Wifi SSID
 * `{ "code": 0, "ssid": "REDACTED", "resp": "Successfully" }`
* `/api/getpasswd` - returns the configured Wifi password
 * `{ "code": 0, "passwd": "REDACTED", "resp": "Successfully" }`
* `/api/version`
 * `{ "code":0, "version":"18:47/2025/11/22", "hwid":"0x2426", "rootfs":"zoom_common" }`
* `/api/productInfo2`
 * `{ "code": 0,"OEM_FPGA_VER":"","OEM_HW_VER":"","OEM_SW_VER":"","OEM_PRODUCT_NUM":"","OEM_SERIL_NUM":"","OEM_MODEL_NAME":"","OEM_RPODUCE_DATE":"","OEM_PREFIX_SSID":"","OEM_MANUFACTURER_NAME":""}`
 
Hitting `/api/pushfile.cgi` gave me this and then crashed the web server, I think. Gonna stop 
poking at these for now, but it looks like you can manipulate the file system through it.

```
> printf "GET /api/pushfile.cgi HTTP/1.1\r\n\r\n" | nc 192.168.11.123 80
HTTP/1.0 200 OK
cgiMain start ------------>
Content-type: text/html

<HTML><HEAD>
<TITLE>cgic test</TITLE></HEAD>
<BODY><H1>cgic test</H1>
<H3> Get filename failed !!!! </H3>
<H3>file_size :0</H3>
<H3>cgiFormFileOpen Failed</H3>
<H3>fileNameOnServer open Failed </H3>
<H3>fileNameOnServer save success ; path:/storage/update/ </H3>
</BODY></HTML>
service send: type = 2, value = 80
```

### FTP server

This requires `root` as username and any password.

It just serves the `/storage` directory, which is where screenshots and video recordings go.

Honestly, not super interesting.

```
# find /storage
/storage/20251116
/storage/20251116/IMG_20251116120335_00.jpeg
/storage/20251116/IMG_20251116120341_00.jpeg
/storage/20251116/IMG_20251116120338_00.jpeg
/storage/20251116/IMG_20251116120340_00.jpeg
/storage/20251111
/storage/20251111/IMG_20251111213413_00.jpeg
/storage/20251111/IMG_20251111215835_00.jpeg
/storage/20251111/IMG_20251111175230_00.jpeg
/storage/20251111/IMG_20251111195309_00.jpeg
/storage/20251111/IMG_20251111220008_00.jpeg
/storage/update
/storage/20251112
/storage/20251112/IMG_20251112174501_00.jpeg
/storage/20251112/IMG_20251112182059_00.jpeg
/storage/20251112/IMG_20251112182058_00.jpeg
/storage/20251112/VID_20251112174450.mp4
/storage/20251112/IMG_20251112174435_00.jpeg
/storage/20251112/IMG_20251112182057_00.jpeg
```

I tried putting a symlink in there to somewhere else in the filesystem to see if I
could traverse outside of `/storage` with it. It didn't work. Shrug, plenty of other ways to do the same.

### SuperUI

SuperUI is probably the thing that runs the whole show. It's a binary in `/usr/bin`:

```
# which superui
/usr/bin/superui
# ls -lah /usr/bin | grep superui
-rwxrwxr-x    1 root     root      580.2K Feb 19  2025 superui
```

There's a directory in the root, `/parameter` that contains config files, including one for SuperUI:

```
# ls -la /parameter
total 20
drwxr-xr-x    3 root     root          1024 Jan  1  2025 .
drwxr-xr-x   22 root     root          1024 Aug 21  2014 ..
-rw-r--r--    1 root     root          1024 Nov 22 18:51 device.cfg
drwx------    2 root     root         12288 Aug 21  2014 lost+found
-rw-r--r--    1 root     root            39 Nov 11 21:36 parameter.cfg
----------    1 root     root            20 Jan  1  2025 product_PN
----------    1 root     root            20 Jan  1  2025 product_SN
-rw-r--r--    1 root     root          1869 Nov 22 18:16 superui.cfg
# cat /parameter/superui.cfg
{
        "version":      12,
        "parameter":    {
                "view_mode":    "HAND_MODE",
                "screen_brightness":    "LEVEL_5",
                "uvc_flag":     "DISABLE",
                "device_polarity":      "RED_HOT",
                "reticle_gun":  "GUN_A",
                "calibration_mode":     "CALIBRATION_AUTO",
                "mic_switch":   "ENABLE",
                "compass_switch":       "ENABLE",
                "wifi_switch":  "ENABLE",
                "bt_switch":    "ENABLE",
                "sleep_flag":   "SLEEP_OFF",
                "state_hiding_switch":  "DISABLE",
                "impact_recording_flag":        "DISABLE",
                "laser_range_flag":     "DISABLE",
                "image_brightness":     "LEVEL_4",
                "image_compare":        "LEVEL_4",
                "image_sharpness":      "LEVEL_5",
                "target_range_unit":    "RANGE_UNIT_M",
                "reticle_type": "RETICLE_TYPE_1",
                "reticle_colour":       "RETICLE_GREEN",
                "reticle_brightness":   "RETICLE_BRIGHTNESS_5",
                "device_video_record_flag":     "DISABLE",
                "reticle_center_x":     0,
                "reticle_center_y":     0,
                "reticle_enable_flag":  "DISABLE",
                "front_mode_flag":      "DISABLE",
                "mx_offset":    33126,
                "my_offset":    32248,
                "mz_offset":    32361,
                "reticle_pos_x":        0,
                "reticle_pos_y":        0,
                "clip_on_win_pos_x":    -6,
                "clip_on_win_pos_y":    14,
                "helmet_win_pos_x":     0,
                "helmet_win_pos_y":     0,
                "distance_flag":        "DISTANCE_100M",
                "color_type":   "COLOR_C",
                "zeroing_distance":     [[100, 200, 300], [100, 200, 300], [100, 200, 300]],
                "zeroing_result":       [[[0, 0], [0, 0], [0, 0]], [[0, 0], [0, 0], [0, 0]], [[0, 0], [0, 0], [0, 0]]]
        }
}}
}100M",
        "color_type": "COLOR_C",
        "zeroing_distance": [
                [100, 200, 300],
                [100, 200, 300],
                [100, 200, 300]
        ],
        "zeroing_result": [
            [
                [0, 0],
                [0, 0],
                [0, 0]
            ],
            [
                [0, 0],
                [0, 0],
                [0, 0]
            ],
            [
                [0, 0],
                [0, 0],
                [0, 0]
            ]
        ]


    }
}
```

Not sure if `uvc_flag` refers to USB Video Class - that is, mounting as a camera over USB - but I
tried to set it to `ENABLE` and rebooted with no obvious change to behavior. The rest appears to be
all of the settings normally available through the menus.

### Loose ends

I didn't dig into the BLE handling or the media streaming server. I believe `test_recoder` is the media
streaming service, and it appears to do hardware based H.264 encoding through a library called
`libinfiray2_hard_x264hwenc.so`.ls /

## Filesystem

List of mounts:

```
# mount
rootfs on / type rootfs (rw)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
/dev/mmcblk0p8 on / type ext4 (rw,relatime,errors=remount-ro,data=ordered)
proc on /proc type proc (rw,relatime)
sysfs on /sys type sysfs (rw,relatime)
tmpfs on /dev type tmpfs (rw,relatime)
tmpfs on /tmp type tmpfs (rw,relatime)
devpts on /dev/pts type devpts (rw,relatime,mode=600,ptmxmode=000)
tmpfs on /dev/shm type tmpfs (rw,relatime)
/dev/mmcblk0p5 on /parameter type ext4 (rw,relatime,data=ordered)
/dev/mmcblk0p9 on /storage type ext4 (rw,noatime,nodiratime,discard,nobarrier)
/dev/mmcblk0p8 on /mnt/sd type ext4 (rw,relatime,errors=remount-ro,data=ordered)
/dev/mmcblk0p7 on /usr/data type ext4 (rw,noatime,nodiratime,discard,nobarrier)
```

It uses a journaling file system, which is good to see on a battery powered device like this.

Here's a list of everything on the filesystem:

```
# find / -path /dev -prune -o -path /sys -prune -o -path /proc -prune -o -print
/
/media
/media/.gitignore
/mnt
/mnt/sd
/mnt/sd/media
/mnt/sd/media/.gitignore
/mnt/sd/mnt
/mnt/sd/mnt/sd
/mnt/sd/mnt/.gitignore
/mnt/sd/proc
/mnt/sd/proc/.gitignore
/mnt/sd/sbin
/mnt/sd/sbin/mkswap
/mnt/sd/sbin/route
/mnt/sd/sbin/iptunnel
/mnt/sd/sbin/mkfs.minix
/mnt/sd/sbin/mkfs.msdos
/mnt/sd/sbin/wifi_connect.sh
/mnt/sd/sbin/logread
/mnt/sd/sbin/adbserver.sh
/mnt/sd/sbin/ipneigh
/mnt/sd/sbin/ipaddr
/mnt/sd/sbin/mkfs.vfat
/mnt/sd/sbin/ifup
/mnt/sd/sbin/devmem
/mnt/sd/sbin/mkfs.ext2
/mnt/sd/sbin/uevent
/mnt/sd/sbin/adjtimex
/mnt/sd/sbin/raidautorun
/mnt/sd/sbin/setconsole
/mnt/sd/sbin/mkdosfs
/mnt/sd/sbin/adbd
/mnt/sd/sbin/makedevs
/mnt/sd/sbin/iwgetid
/mnt/sd/sbin/iwpriv
/mnt/sd/sbin/tunctl
/mnt/sd/sbin/klogd
/mnt/sd/sbin/swapoff
/mnt/sd/sbin/ap_wifi_sta.sh
/mnt/sd/sbin/ip
/mnt/sd/sbin/findfs
/mnt/sd/sbin/fstrim
/mnt/sd/sbin/lsmod
/mnt/sd/sbin/vconfig
/mnt/sd/sbin/ifconfig
/mnt/sd/sbin/update
/mnt/sd/sbin/blkid
/mnt/sd/sbin/getpackage
/mnt/sd/sbin/switch_root
/mnt/sd/sbin/acpid
/mnt/sd/sbin/fsck.minix
/mnt/sd/sbin/hwclock
/mnt/sd/sbin/iprule
/mnt/sd/sbin/mkfs.fat
/mnt/sd/sbin/mount.lowntfs-3g
/mnt/sd/sbin/init
/mnt/sd/sbin/wifi_start
/mnt/sd/sbin/swapon
/mnt/sd/sbin/loadkmap
/mnt/sd/sbin/getty
/mnt/sd/sbin/iwconfig
/mnt/sd/sbin/depmod
/mnt/sd/sbin/fbsplash
/mnt/sd/sbin/iproute
/mnt/sd/sbin/blockdev
/mnt/sd/sbin/zcip
/mnt/sd/sbin/pivot_root
/mnt/sd/sbin/sysctl
/mnt/sd/sbin/arp
/mnt/sd/sbin/reboot
/mnt/sd/sbin/modprobe
/mnt/sd/sbin/ifenslave
/mnt/sd/sbin/rmmod
/mnt/sd/sbin/bootchartd
/mnt/sd/sbin/syslogd
/mnt/sd/sbin/iwspy
/mnt/sd/sbin/runlevel
/mnt/sd/sbin/akiss
/mnt/sd/sbin/airkiss
/mnt/sd/sbin/watchdog
/mnt/sd/sbin/slattach
/mnt/sd/sbin/mod_init.sh
/mnt/sd/sbin/iplink
/mnt/sd/sbin/sulogin
/mnt/sd/sbin/mount.ntfs-3g
/mnt/sd/sbin/freeramdisk
/mnt/sd/sbin/modinfo
/mnt/sd/sbin/ifdown
/mnt/sd/sbin/nameif
/mnt/sd/sbin/losetup
/mnt/sd/sbin/hdparm
/mnt/sd/sbin/insmod
/mnt/sd/sbin/mdev
/mnt/sd/sbin/iwlist
/mnt/sd/sbin/udhcpc
/mnt/sd/sbin/mke2fs
/mnt/sd/sbin/fdisk
/mnt/sd/sbin/halt
/mnt/sd/sbin/wifi_stop.sh
/mnt/sd/sbin/start-stop-daemon
/mnt/sd/sbin/sta_wifi_ap.sh
/mnt/sd/sbin/mod_install_all.sh
/mnt/sd/sbin/poweroff
/mnt/sd/bin
/mnt/sd/bin/dumpkmap
/mnt/sd/bin/mv
/mnt/sd/bin/setarch
/mnt/sd/bin/rm
/mnt/sd/bin/rmdir
/mnt/sd/bin/catv
/mnt/sd/bin/ping6
/mnt/sd/bin/egrep
/mnt/sd/bin/fatattr
/mnt/sd/bin/hostname
/mnt/sd/bin/mpstat
/mnt/sd/bin/sed
/mnt/sd/bin/rev
/mnt/sd/bin/gzip
/mnt/sd/bin/ln
/mnt/sd/bin/chgrp
/mnt/sd/bin/ash
/mnt/sd/bin/getopt
/mnt/sd/bin/bash
/mnt/sd/bin/umount
/mnt/sd/bin/pwd
/mnt/sd/bin/sync
/mnt/sd/bin/busybox
/mnt/sd/bin/true
/mnt/sd/bin/dnsdomainname
/mnt/sd/bin/gunzip
/mnt/sd/bin/watch
/mnt/sd/bin/dmesg
/mnt/sd/bin/grep
/mnt/sd/bin/date
/mnt/sd/bin/conspy
/mnt/sd/bin/ping
/mnt/sd/bin/base64
/mnt/sd/bin/mktemp
/mnt/sd/bin/kbd_mode
/mnt/sd/bin/zcat
/mnt/sd/bin/tar
/mnt/sd/bin/ls
/mnt/sd/bin/false
/mnt/sd/bin/chown
/mnt/sd/bin/sh
/mnt/sd/bin/nice
/mnt/sd/bin/hush
/mnt/sd/bin/ps
/mnt/sd/bin/stty
/mnt/sd/bin/setserial
/mnt/sd/bin/cttyhack
/mnt/sd/bin/makemime
/mnt/sd/bin/more
/mnt/sd/bin/fdflush
/mnt/sd/bin/stat
/mnt/sd/bin/kill
/mnt/sd/bin/mt
/mnt/sd/bin/pidof
/mnt/sd/bin/cat
/mnt/sd/bin/scriptreplay
/mnt/sd/bin/printenv
/mnt/sd/bin/fgrep
/mnt/sd/bin/chmod
/mnt/sd/bin/reformime
/mnt/sd/bin/run-parts
/mnt/sd/bin/cpio
/mnt/sd/bin/pipe_progress
/mnt/sd/bin/dd
/mnt/sd/bin/fsync
/mnt/sd/bin/vi
/mnt/sd/bin/ionice
/mnt/sd/bin/su
/mnt/sd/bin/df
/mnt/sd/bin/mount
/mnt/sd/bin/usleep
/mnt/sd/bin/mountpoint
/mnt/sd/bin/echo
/mnt/sd/bin/netstat
/mnt/sd/bin/sleep
/mnt/sd/bin/touch
/mnt/sd/bin/uname
/mnt/sd/bin/ipcalc
/mnt/sd/bin/mknod
/mnt/sd/bin/ed
/mnt/sd/bin/login
/mnt/sd/bin/linux64
/mnt/sd/bin/cp
/mnt/sd/bin/mkdir
/mnt/sd/bin/iostat
/mnt/sd/bin/linux32
/mnt/sd/run
/mnt/sd/run/udhcpd.pid
/mnt/sd/run/syslogd.pid
/mnt/sd/run/network
/mnt/sd/run/telnetd.pid
/mnt/sd/run/klogd.pid
/mnt/sd/run/utmp
/mnt/sd/run/dbus
/mnt/sd/run/dbus/system_bus_socket
/mnt/sd/run/dbus/pid
/mnt/sd/run/ifstate
/mnt/sd/run/blkid
/mnt/sd/run/blkid/blkid.tab
/mnt/sd/run/readme
/mnt/sd/etc
/mnt/sd/etc/inputrc
/mnt/sd/etc/os-release
/mnt/sd/etc/httpd.conf
/mnt/sd/etc/nsswitch.conf
/mnt/sd/etc/media_codecs_google_video.xml
/mnt/sd/etc/inittab
/mnt/sd/etc/hosts
/mnt/sd/etc/network
/mnt/sd/etc/network/if-pre-up.d
/mnt/sd/etc/network/if-pre-up.d/wait_iface
/mnt/sd/etc/network/interfaces
/mnt/sd/etc/ota_res
/mnt/sd/etc/ota_res/update.sh
/mnt/sd/etc/ota_res/recovery.conf
/mnt/sd/etc/ota_res/key
/mnt/sd/etc/ota_res/key/key.pub
/mnt/sd/etc/ota_res/recovery
/mnt/sd/etc/ota_res/update_stage.sh
/mnt/sd/etc/ota_res/update_play.sh
/mnt/sd/etc/media_codecs.xml
/mnt/sd/etc/ld.so.cache
/mnt/sd/etc/hostapd.deny
/mnt/sd/etc/resolv.conf
/mnt/sd/etc/appenv.sh
/mnt/sd/etc/mke2fs.conf
/mnt/sd/etc/random-seed
/mnt/sd/etc/usb
/mnt/sd/etc/usb/usb_removing
/mnt/sd/etc/usb/usb_inserting
/mnt/sd/etc/dmesg_log.sh
/mnt/sd/etc/mixer_paths.xml
/mnt/sd/etc/fstab
/mnt/sd/etc/init.d
/mnt/sd/etc/init.d/S40copy_ota_res
/mnt/sd/etc/init.d/S50telnet
/mnt/sd/etc/init.d/S20parameter
/mnt/sd/etc/init.d/S20urandom
/mnt/sd/etc/init.d/S40network
/mnt/sd/etc/init.d/S22debuggerd
/mnt/sd/etc/init.d/S90dlog
/mnt/sd/etc/init.d/rcS
/mnt/sd/etc/init.d/S10mdev
/mnt/sd/etc/init.d/TS90misc
/mnt/sd/etc/init.d/S30dbus
/mnt/sd/etc/init.d/adb
/mnt/sd/etc/init.d/adb/S310adb
/mnt/sd/etc/init.d/adb/S440adb
/mnt/sd/etc/init.d/S12storage
/mnt/sd/etc/init.d/S01logging
/mnt/sd/etc/init.d/S91app
/mnt/sd/etc/init.d/S21data
/mnt/sd/etc/init.d/S90misc
/mnt/sd/etc/init.d/S20mntsd
/mnt/sd/etc/init.d/S90adb
/mnt/sd/etc/init.d/rcK
/mnt/sd/etc/bluetooth
/mnt/sd/etc/bluetooth/rfcomm.conf
/mnt/sd/etc/bluetooth/main.conf
/mnt/sd/etc/wpa_supplicant.conf
/mnt/sd/etc/media_profiles.xml
/mnt/sd/etc/protocols
/mnt/sd/etc/services
/mnt/sd/etc/superui.cfg
/mnt/sd/etc/profile.d
/mnt/sd/etc/profile.d/superui.sh
/mnt/sd/etc/profile.d/umask.sh
/mnt/sd/etc/profile.d/debugger.sh
/mnt/sd/etc/group
/mnt/sd/etc/passwd
/mnt/sd/etc/ssl
/mnt/sd/etc/ssl/openssl.cnf
/mnt/sd/etc/media_codecs_google_audio.xml
/mnt/sd/etc/shadow
/mnt/sd/etc/hostapd.conf
/mnt/sd/etc/mtab
/mnt/sd/etc/udhcpd.conf
/mnt/sd/etc/sudoers
/mnt/sd/etc/mdev.conf
/mnt/sd/etc/profile
/mnt/sd/etc/dbus-1
/mnt/sd/etc/dbus-1/system.d
/mnt/sd/etc/dbus-1/system.d/pulseaudio-system.conf
/mnt/sd/etc/dbus-1/system.d/wpa_supplicant.conf
/mnt/sd/etc/dbus-1/system.d/bluetooth.conf
/mnt/sd/etc/dbus-1/session.conf
/mnt/sd/etc/dbus-1/system.conf
/mnt/sd/etc/audio_policy.conf
/mnt/sd/etc/libnl
/mnt/sd/etc/libnl/classid
/mnt/sd/etc/libnl/pktloc
/mnt/sd/tmp
/mnt/sd/tmp/ldconfig
/mnt/sd/tmp/ldconfig/aux-cache
/mnt/sd/tmp/.gitignore
/mnt/sd/linuxrc
/mnt/sd/root
/mnt/sd/root/.gitignore
/mnt/sd/root/.ash_history
/mnt/sd/lib
/mnt/sd/lib/libmount.so.1.1.0
/mnt/sd/lib/libblkid.so.1
/mnt/sd/lib/libuuid.so.1
/mnt/sd/lib/libdl.so.2
/mnt/sd/lib/libcrypt-2.22.so
/mnt/sd/lib/libatomic.so.1
/mnt/sd/lib/libblkid.so.1.1.0
/mnt/sd/lib/libthread_db-1.0.so
/mnt/sd/lib/libnsl.so.1
/mnt/sd/lib/libnss_files.so.2
/mnt/sd/lib/libnsl-2.22.so
/mnt/sd/lib/libm-2.22.so
/mnt/sd/lib/libc.so.6
/mnt/sd/lib/libgcc_s.so.1
/mnt/sd/lib/libm.so.6
/mnt/sd/lib/librt-2.22.so
/mnt/sd/lib/ld.so.1
/mnt/sd/lib/libnss_files-2.22.so
/mnt/sd/lib/libuuid.so.1.3.0
/mnt/sd/lib/libutil-2.22.so
/mnt/sd/lib/libnss_dns.so.2
/mnt/sd/lib/libmount.so.1
/mnt/sd/lib/firmware
/mnt/sd/lib/firmware/BCM4343A1_001.002.009.0025.0059.hcd
/mnt/sd/lib/libanl.so.1
/mnt/sd/lib/libcrypt.so.1
/mnt/sd/lib/bash
/mnt/sd/lib/bash/printenv
/mnt/sd/lib/bash/rmdir
/mnt/sd/lib/bash/print
/mnt/sd/lib/bash/uname
/mnt/sd/lib/bash/logname
/mnt/sd/lib/bash/strftime
/mnt/sd/lib/bash/head
/mnt/sd/lib/bash/whoami
/mnt/sd/lib/bash/ln
/mnt/sd/lib/bash/tty
/mnt/sd/lib/bash/Makefile.inc
/mnt/sd/lib/bash/sleep
/mnt/sd/lib/bash/mkdir
/mnt/sd/lib/bash/realpath
/mnt/sd/lib/bash/setpgid
/mnt/sd/lib/bash/pathchk
/mnt/sd/lib/bash/sync
/mnt/sd/lib/bash/unlink
/mnt/sd/lib/bash/id
/mnt/sd/lib/bash/dirname
/mnt/sd/lib/bash/tee
/mnt/sd/lib/bash/mypid
/mnt/sd/lib/bash/basename
/mnt/sd/lib/bash/finfo
/mnt/sd/lib/bash/push
/mnt/sd/lib/bash/truefalse
/mnt/sd/lib/libc-2.22.so
/mnt/sd/lib/libdl-2.22.so
/mnt/sd/lib/libanl-2.22.so
/mnt/sd/lib/libpthread.so.0
/mnt/sd/lib/ld-2.22.so
/mnt/sd/lib/libutil.so.1
/mnt/sd/lib/libatomic.so.1.1.0
/mnt/sd/lib/libresolv.so.2
/mnt/sd/lib/libnss_dns-2.22.so
/mnt/sd/lib/pkgconfig
/mnt/sd/lib/pkgconfig/bash.pc
/mnt/sd/lib/libpthread-2.22.so
/mnt/sd/lib/libresolv-2.22.so
/mnt/sd/lib/libthread_db.so.1
/mnt/sd/lib/librt.so.1
/mnt/sd/firmware
/mnt/sd/firmware/wifimac.txt
/mnt/sd/firmware/nv_43438_a1.cal
/mnt/sd/firmware/fw_43438_a1.bin
/mnt/sd/parameter
/mnt/sd/opt
/mnt/sd/opt/.gitignore
/mnt/sd/lost+found
/mnt/sd/usr
/mnt/sd/usr/libexec
/mnt/sd/usr/libexec/dbus-daemon-launch-helper
/mnt/sd/usr/libexec/sudo
/mnt/sd/usr/libexec/sudo/libsudo_util.so
/mnt/sd/usr/libexec/sudo/sudoers.so
/mnt/sd/usr/libexec/sudo/sudo_noexec.so
/mnt/sd/usr/libexec/sudo/group_file.so
/mnt/sd/usr/libexec/sudo/libsudo_util.so.0.0.0
/mnt/sd/usr/libexec/sudo/libsudo_util.so.0
/mnt/sd/usr/libexec/sudo/system_group.so
/mnt/sd/usr/sbin
/mnt/sd/usr/sbin/delgroup
/mnt/sd/usr/sbin/svlogd
/mnt/sd/usr/sbin/bspatch
/mnt/sd/usr/sbin/mkfs.ext4dev
/mnt/sd/usr/sbin/ubirsvol
/mnt/sd/usr/sbin/ip6tables-restore
/mnt/sd/usr/sbin/sendmail
/mnt/sd/usr/sbin/chpasswd
/mnt/sd/usr/sbin/fbset
/mnt/sd/usr/sbin/ubirename
/mnt/sd/usr/sbin/popmaildir
/mnt/sd/usr/sbin/httpd
/mnt/sd/usr/sbin/nbd-client
/mnt/sd/usr/sbin/ubiformat
/mnt/sd/usr/sbin/ntpd
/mnt/sd/usr/sbin/mkfs.ext2
/mnt/sd/usr/sbin/bluetoothd
/mnt/sd/usr/sbin/lpd
/mnt/sd/usr/sbin/hciconfig
/mnt/sd/usr/sbin/chroot
/mnt/sd/usr/sbin/rdev
/mnt/sd/usr/sbin/rtcwake
/mnt/sd/usr/sbin/tftpd
/mnt/sd/usr/sbin/readahead
/mnt/sd/usr/sbin/chat
/mnt/sd/usr/sbin/loadfont
/mnt/sd/usr/sbin/flash_unlock
/mnt/sd/usr/sbin/ubimkvol
/mnt/sd/usr/sbin/i2cset
/mnt/sd/usr/sbin/ip6tables-save
/mnt/sd/usr/sbin/ubiupdatevol
/mnt/sd/usr/sbin/killall5
/mnt/sd/usr/sbin/nandtest
/mnt/sd/usr/sbin/ubicrc32
/mnt/sd/usr/sbin/telnetd
/mnt/sd/usr/sbin/hciattach
/mnt/sd/usr/sbin/ip6tables
/mnt/sd/usr/sbin/ether-wake
/mnt/sd/usr/sbin/i2cget
/mnt/sd/usr/sbin/dnsd
/mnt/sd/usr/sbin/bsdiff
/mnt/sd/usr/sbin/remove-shell
/mnt/sd/usr/sbin/adduser
/mnt/sd/usr/sbin/visudo
/mnt/sd/usr/sbin/e4crypt
/mnt/sd/usr/sbin/wpa_supplicant
/mnt/sd/usr/sbin/crond
/mnt/sd/usr/sbin/ubirmvol
/mnt/sd/usr/sbin/m200_wifi_sta.sh
/mnt/sd/usr/sbin/flash_lock
/mnt/sd/usr/sbin/ubiattach
/mnt/sd/usr/sbin/xtables-multi
/mnt/sd/usr/sbin/mtd_debug
/mnt/sd/usr/sbin/fakeidentd
/mnt/sd/usr/sbin/setfont
/mnt/sd/usr/sbin/dhcprelay
/mnt/sd/usr/sbin/ubiblock
/mnt/sd/usr/sbin/i2cdetect
/mnt/sd/usr/sbin/ifplugd
/mnt/sd/usr/sbin/flashcp
/mnt/sd/usr/sbin/readprofile
/mnt/sd/usr/sbin/addgroup
/mnt/sd/usr/sbin/mkfs.jffs2
/mnt/sd/usr/sbin/mkfs.ext3
/mnt/sd/usr/sbin/ubinfo
/mnt/sd/usr/sbin/nandwrite
/mnt/sd/usr/sbin/ftpd
/mnt/sd/usr/sbin/ftl_check
/mnt/sd/usr/sbin/recovery
/mnt/sd/usr/sbin/iptables-save
/mnt/sd/usr/sbin/inetd
/mnt/sd/usr/sbin/i2cdump
/mnt/sd/usr/sbin/mkfs.ext4
/mnt/sd/usr/sbin/arping
/mnt/sd/usr/sbin/nanddump
/mnt/sd/usr/sbin/deluser
/mnt/sd/usr/sbin/add-shell
/mnt/sd/usr/sbin/brctl
/mnt/sd/usr/sbin/iptables-restore
/mnt/sd/usr/sbin/fdformat
/mnt/sd/usr/sbin/ubinize
/mnt/sd/usr/sbin/flash_erase
/mnt/sd/usr/sbin/ubidetach
/mnt/sd/usr/sbin/powertop
/mnt/sd/usr/sbin/mke2fs
/mnt/sd/usr/sbin/rdate
/mnt/sd/usr/sbin/setlogcons
/mnt/sd/usr/sbin/hostapd
/mnt/sd/usr/sbin/iptables
/mnt/sd/usr/sbin/udhcpd
/mnt/sd/usr/sbin/mtdinfo
/mnt/sd/usr/bin
/mnt/sd/usr/bin/unxz
/mnt/sd/usr/bin/dbus-uuidgen
/mnt/sd/usr/bin/dbus-cleanup-sockets
/mnt/sd/usr/bin/ircp
/mnt/sd/usr/bin/yes
/mnt/sd/usr/bin/pcregrep
/mnt/sd/usr/bin/reset
/mnt/sd/usr/bin/gapplication
/mnt/sd/usr/bin/uuencode
/mnt/sd/usr/bin/mesg
/mnt/sd/usr/bin/expr
/mnt/sd/usr/bin/ble
/mnt/sd/usr/bin/nsenter
/mnt/sd/usr/bin/nohup
/mnt/sd/usr/bin/tinycap
/mnt/sd/usr/bin/dlog_logger
/mnt/sd/usr/bin/sdl2-config
/mnt/sd/usr/bin/lspci
/mnt/sd/usr/bin/lpq
/mnt/sd/usr/bin/update_checking
/mnt/sd/usr/bin/obex_test
/mnt/sd/usr/bin/dos2unix
/mnt/sd/usr/bin/ntfs-3g.secaudit
/mnt/sd/usr/bin/groups
/mnt/sd/usr/bin/logname
/mnt/sd/usr/bin/traceroute6
/mnt/sd/usr/bin/stateMachineServer
/mnt/sd/usr/bin/resize
/mnt/sd/usr/bin/less
/mnt/sd/usr/bin/ar
/mnt/sd/usr/bin/uptime
/mnt/sd/usr/bin/whois
/mnt/sd/usr/bin/last
/mnt/sd/usr/bin/hostapd_cli
/mnt/sd/usr/bin/lzcat
/mnt/sd/usr/bin/rfcomm
/mnt/sd/usr/bin/screen_out_test
/mnt/sd/usr/bin/dbus-test-tool
/mnt/sd/usr/bin/man
/mnt/sd/usr/bin/install
/mnt/sd/usr/bin/strace-log-merge
/mnt/sd/usr/bin/pcretest
/mnt/sd/usr/bin/tftp
/mnt/sd/usr/bin/uniq
/mnt/sd/usr/bin/bzip2
/mnt/sd/usr/bin/arecord
/mnt/sd/usr/bin/ldconfig
/mnt/sd/usr/bin/fgconsole
/mnt/sd/usr/bin/tinymix
/mnt/sd/usr/bin/lowntfs-3g
/mnt/sd/usr/bin/setuidgid
/mnt/sd/usr/bin/agent
/mnt/sd/usr/bin/i2ctransfer
/mnt/sd/usr/bin/mkfifo
/mnt/sd/usr/bin/pgrep
/mnt/sd/usr/bin/svc
/mnt/sd/usr/bin/md5sum
/mnt/sd/usr/bin/envuidgid
/mnt/sd/usr/bin/bluetoothctl
/mnt/sd/usr/bin/clear
/mnt/sd/usr/bin/chrt
/mnt/sd/usr/bin/head
/mnt/sd/usr/bin/nmeter
/mnt/sd/usr/bin/bt_enable_a0
/mnt/sd/usr/bin/du
/mnt/sd/usr/bin/ble_server
/mnt/sd/usr/bin/sum
/mnt/sd/usr/bin/dbus-send
/mnt/sd/usr/bin/pkill
/mnt/sd/usr/bin/wpa_supplicant
/mnt/sd/usr/bin/sort
/mnt/sd/usr/bin/whoami
/mnt/sd/usr/bin/uudecode
/mnt/sd/usr/bin/awk
/mnt/sd/usr/bin/cut
/mnt/sd/usr/bin/gdbserver
/mnt/sd/usr/bin/dpkg
/mnt/sd/usr/bin/ttysize
/mnt/sd/usr/bin/logger
/mnt/sd/usr/bin/dbus-monitor
/mnt/sd/usr/bin/ftpput
/mnt/sd/usr/bin/sudoreplay
/mnt/sd/usr/bin/gresource
/mnt/sd/usr/bin/bt_enable
/mnt/sd/usr/bin/cryptpw
/mnt/sd/usr/bin/deallocvt
/mnt/sd/usr/bin/eject
/mnt/sd/usr/bin/dpkg-split
/mnt/sd/usr/bin/split
/mnt/sd/usr/bin/sudo
/mnt/sd/usr/bin/RTSPServer
/mnt/sd/usr/bin/readlink
/mnt/sd/usr/bin/ntfs-3g.probe
/mnt/sd/usr/bin/lsusb
/mnt/sd/usr/bin/unzip
/mnt/sd/usr/bin/udc_mass_storage.sh
/mnt/sd/usr/bin/killall
/mnt/sd/usr/bin/beep
/mnt/sd/usr/bin/dpkg-deb
/mnt/sd/usr/bin/script
/mnt/sd/usr/bin/nslookup
/mnt/sd/usr/bin/dbus-update-activation-environment
/mnt/sd/usr/bin/softlimit
/mnt/sd/usr/bin/unlzma
/mnt/sd/usr/bin/dlogutil
/mnt/sd/usr/bin/tty
/mnt/sd/usr/bin/runsv
/mnt/sd/usr/bin/openvt
/mnt/sd/usr/bin/aserver
/mnt/sd/usr/bin/dlogctrl
/mnt/sd/usr/bin/sha3sum
/mnt/sd/usr/bin/who
/mnt/sd/usr/bin/wc
/mnt/sd/usr/bin/irobex_palm3
/mnt/sd/usr/bin/pscan
/mnt/sd/usr/bin/hexdump
/mnt/sd/usr/bin/uyuv_to_nv12
/mnt/sd/usr/bin/env
/mnt/sd/usr/bin/vlock
/mnt/sd/usr/bin/pmap
/mnt/sd/usr/bin/traceroute
/mnt/sd/usr/bin/hd
/mnt/sd/usr/bin/setsid
/mnt/sd/usr/bin/dumpleases
/mnt/sd/usr/bin/tcpsvd
/mnt/sd/usr/bin/dbus-daemon
/mnt/sd/usr/bin/printf
/mnt/sd/usr/bin/blkdiscard
/mnt/sd/usr/bin/cal
/mnt/sd/usr/bin/openssl
/mnt/sd/usr/bin/smemcap
/mnt/sd/usr/bin/realpath
/mnt/sd/usr/bin/envdir
/mnt/sd/usr/bin/servicemanager
/mnt/sd/usr/bin/tail
/mnt/sd/usr/bin/showkey
/mnt/sd/usr/bin/wget
/mnt/sd/usr/bin/od
/mnt/sd/usr/bin/ftpget
/mnt/sd/usr/bin/tinyplay
/mnt/sd/usr/bin/sv
/mnt/sd/usr/bin/aplay
/mnt/sd/usr/bin/lpr
/mnt/sd/usr/bin/iptables-xml
/mnt/sd/usr/bin/gio
/mnt/sd/usr/bin/sdptool
/mnt/sd/usr/bin/gsettings
/mnt/sd/usr/bin/strings
/mnt/sd/usr/bin/microcom
/mnt/sd/usr/bin/lsof
/mnt/sd/usr/bin/TakePicture
/mnt/sd/usr/bin/ntfs-3g.usermap
/mnt/sd/usr/bin/wpa_cli
/mnt/sd/usr/bin/xz
/mnt/sd/usr/bin/gdbus
/mnt/sd/usr/bin/xargs
/mnt/sd/usr/bin/ciptool
/mnt/sd/usr/bin/gatttool
/mnt/sd/usr/bin/irxfer
/mnt/sd/usr/bin/telnet
/mnt/sd/usr/bin/find
/mnt/sd/usr/bin/time
/mnt/sd/usr/bin/tree
/mnt/sd/usr/bin/mediaserver
/mnt/sd/usr/bin/brcm_patchram_plus
/mnt/sd/usr/bin/[
/mnt/sd/usr/bin/wifiserver
/mnt/sd/usr/bin/rx
/mnt/sd/usr/bin/pminstall
/mnt/sd/usr/bin/sha512sum
/mnt/sd/usr/bin/ntfs-3g
/mnt/sd/usr/bin/gio-querymodules
/mnt/sd/usr/bin/xzcat
/mnt/sd/usr/bin/VideoRecord
/mnt/sd/usr/bin/chvt
/mnt/sd/usr/bin/dbus-run-session
/mnt/sd/usr/bin/sudoedit
/mnt/sd/usr/bin/volname
/mnt/sd/usr/bin/which
/mnt/sd/usr/bin/start-stop-daemon
/mnt/sd/usr/bin/tinypcminfo
/mnt/sd/usr/bin/sha1sum
/mnt/sd/usr/bin/bzcat
/mnt/sd/usr/bin/AudioRecord
/mnt/sd/usr/bin/test-debugger
/mnt/sd/usr/bin/jpegtran
/mnt/sd/usr/bin/amixer
/mnt/sd/usr/bin/ipcrm
/mnt/sd/usr/bin/unshare
/mnt/sd/usr/bin/udpsvd
/mnt/sd/usr/bin/l2ping
/mnt/sd/usr/bin/dbus-launch
/mnt/sd/usr/bin/passwd
/mnt/sd/usr/bin/runsvdir
/mnt/sd/usr/bin/[[
/mnt/sd/usr/bin/wrjpgcom
/mnt/sd/usr/bin/crontab
/mnt/sd/usr/bin/tr
/mnt/sd/usr/bin/WifiTest
/mnt/sd/usr/bin/unlink
/mnt/sd/usr/bin/bt_enable_a1
/mnt/sd/usr/bin/rdjpgcom
/mnt/sd/usr/bin/strace
/mnt/sd/usr/bin/setkeycodes
/mnt/sd/usr/bin/cksum
/mnt/sd/usr/bin/udhcpc6
/mnt/sd/usr/bin/patch
/mnt/sd/usr/bin/sqlite3
/mnt/sd/usr/bin/udc_mass_storage
/mnt/sd/usr/bin/timeout
/mnt/sd/usr/bin/MediaPlayer
/mnt/sd/usr/bin/cjpeg
/mnt/sd/usr/bin/hcitool
/mnt/sd/usr/bin/chpst
/mnt/sd/usr/bin/fold
/mnt/sd/usr/bin/mkpasswd
/mnt/sd/usr/bin/hostapd
/mnt/sd/usr/bin/hostapd/wlan0
/mnt/sd/usr/bin/mtpserver
/mnt/sd/usr/bin/pstree
/mnt/sd/usr/bin/ipcs
/mnt/sd/usr/bin/seq
/mnt/sd/usr/bin/debuggerd
/mnt/sd/usr/bin/test_recoder
/mnt/sd/usr/bin/sha256sum
/mnt/sd/usr/bin/dc
/mnt/sd/usr/bin/djpeg
/mnt/sd/usr/bin/nc
/mnt/sd/usr/bin/obex_tcp
/mnt/sd/usr/bin/superui
/mnt/sd/usr/bin/id
/mnt/sd/usr/bin/dirname
/mnt/sd/usr/bin/users
/mnt/sd/usr/bin/unix2dos
/mnt/sd/usr/bin/truncate
/mnt/sd/usr/bin/hostid
/mnt/sd/usr/bin/test
/mnt/sd/usr/bin/pwdx
/mnt/sd/usr/bin/fuser
/mnt/sd/usr/bin/tee
/mnt/sd/usr/bin/cmp
/mnt/sd/usr/bin/diff
/mnt/sd/usr/bin/basename
/mnt/sd/usr/bin/renice
/mnt/sd/usr/bin/free
/mnt/sd/usr/bin/lzma
/mnt/sd/usr/bin/flock
/mnt/sd/usr/bin/wall
/mnt/sd/usr/bin/top
/mnt/sd/usr/etc
/mnt/sd/usr/etc/dlog.conf
/mnt/sd/usr/etc/virtual_uart.sh
/mnt/sd/usr/etc/dlog_logger.conf
/mnt/sd/usr/icu
/mnt/sd/usr/icu/icudt55l.dat
/mnt/sd/usr/lib
/mnt/sd/usr/lib/libGAL.so
/mnt/sd/usr/lib/libstagefright_hard_alume.so
/mnt/sd/usr/lib/libe2p.so.2.3
/mnt/sd/usr/lib/libe2p.so
/mnt/sd/usr/lib/libnl-xfrm-3.so.200.22.0
/mnt/sd/usr/lib/libntfs-3g.so
/mnt/sd/usr/lib/libglib-2.0.so.0.5000.2
/mnt/sd/usr/lib/libdbus-1.so.3.14.10
/mnt/sd/usr/lib/libmp4v2.so.2.0.0
/mnt/sd/usr/lib/libntfs-3g.so.87
/mnt/sd/usr/lib/libstagefright_soft_amrwbenc.so
/mnt/sd/usr/lib/libpcreposix.so.0
/mnt/sd/usr/lib/libnettle.so.6.4
/mnt/sd/usr/lib/libgmp.so
/mnt/sd/usr/lib/libnl-route-3.so.200.22.0
/mnt/sd/usr/lib/libpcre.so.1
/mnt/sd/usr/lib/libgui.so
/mnt/sd/usr/lib/libhogweed.so.4.4
/mnt/sd/usr/lib/libsonic.so
/mnt/sd/usr/lib/libstagefright_wfd.so
/mnt/sd/usr/lib/libtinyalsa.so.1.1.0
/mnt/sd/usr/lib/libopenobex.so.1.4.1
/mnt/sd/usr/lib/libpng.so
/mnt/sd/usr/lib/libstagefright_soft_amrdec.so
/mnt/sd/usr/lib/libSDL2_ttf-2.0.so.0
/mnt/sd/usr/lib/libgobject-2.0.so.0
/mnt/sd/usr/lib/libpcreposix.so
/mnt/sd/usr/lib/libtasn1.so
/mnt/sd/usr/lib/liblzo2.so
/mnt/sd/usr/lib/libcommon_time_client.so
/mnt/sd/usr/lib/libatomic.so.1
/mnt/sd/usr/lib/libVIVANTE.so
/mnt/sd/usr/lib/libxtables.so.12
/mnt/sd/usr/lib/libgthread-2.0.so
/mnt/sd/usr/lib/libmtp.so
/mnt/sd/usr/lib/libstagefright_amrnb_common.so
/mnt/sd/usr/lib/libz.so.1.2.8
/mnt/sd/usr/lib/libgobject-2.0.so.0.5000.2
/mnt/sd/usr/lib/libstdc++.so.6.0.21
/mnt/sd/usr/lib/libss.so.2
/mnt/sd/usr/lib/libpcrecpp.so.0
/mnt/sd/usr/lib/libnl-route-3.so
/mnt/sd/usr/lib/libstagefright_soft_aacenc.so
/mnt/sd/usr/lib/libnettle.so.6
/mnt/sd/usr/lib/libwpa_client.so
/mnt/sd/usr/lib/libjpeg.so.8.0.2
/mnt/sd/usr/lib/libhistory.so.7
/mnt/sd/usr/lib/libgnutls-openssl.so.27.0.2
/mnt/sd/usr/lib/libpcre.so.1.2.8
/mnt/sd/usr/lib/libstagefright_yuv.so
/mnt/sd/usr/lib/libcheck.so.0
/mnt/sd/usr/lib/libserviceutility.so
/mnt/sd/usr/lib/libreadline.so
/mnt/sd/usr/lib/libpcreposix.so.0.0.4
/mnt/sd/usr/lib/libcom_err.so.2.1
/mnt/sd/usr/lib/libstagefright_soft_aacdec.so
/mnt/sd/usr/lib/libnbaio.so
/mnt/sd/usr/lib/libz.so.1.2.11
/mnt/sd/usr/lib/libz.so.1
/mnt/sd/usr/lib/libpcrecpp.so
/mnt/sd/usr/lib/libstagefright_foundation.so
/mnt/sd/usr/lib/libVSC.so
/mnt/sd/usr/lib/libsqlite3.so.0.8.6
/mnt/sd/usr/lib/libnl-xfrm-3.so
/mnt/sd/usr/lib/libgio-2.0.so.0.5000.2
/mnt/sd/usr/lib/libgnutlsxx.so.28
/mnt/sd/usr/lib/libpanel.so.5
/mnt/sd/usr/lib/libsqlite3.so.0
/mnt/sd/usr/lib/libform.so
/mnt/sd/usr/lib/libbluetooth.so.3
/mnt/sd/usr/lib/libexpat.so.1
/mnt/sd/usr/lib/libdrmframework.so
/mnt/sd/usr/lib/libturbojpeg.so
/mnt/sd/usr/lib/libform.so.5
/mnt/sd/usr/lib/libnl-genl-3.so.200.22.0
/mnt/sd/usr/lib/libnetutils.so
/mnt/sd/usr/lib/libsync.so
/mnt/sd/usr/lib/libgalUtil.so
/mnt/sd/usr/lib/libSDL2-2.0.so.0
/mnt/sd/usr/lib/libbluetooth.so
/mnt/sd/usr/lib/libdebugger.so
/mnt/sd/usr/lib/libext2fs.so.2
/mnt/sd/usr/lib/libstagefright_httplive.so
/mnt/sd/usr/lib/libGLSLC.so
/mnt/sd/usr/lib/libpanel.so
/mnt/sd/usr/lib/libSDL2_image-2.0.so.0
/mnt/sd/usr/lib/libcrypto.so.1.1
/mnt/sd/usr/lib/libtinyalsa.so
/mnt/sd/usr/lib/libjpeg.so
/mnt/sd/usr/lib/libpng.so.3.37.0
/mnt/sd/usr/lib/libmediautils.so
/mnt/sd/usr/lib/libstagefright_omx.so
/mnt/sd/usr/lib/libnl-nf-3.so.200.22.0
/mnt/sd/usr/lib/libstagefright_soft_vorbisdec.so
/mnt/sd/usr/lib/libstagefright_soft_rawdec.so
/mnt/sd/usr/lib/libxtables.so.12.0.0
/mnt/sd/usr/lib/libglib-2.0.so.0
/mnt/sd/usr/lib/libffi.so.6.0.4
/mnt/sd/usr/lib/libip6tc.so
/mnt/sd/usr/lib/libgmodule-2.0.so
/mnt/sd/usr/lib/libnettle.so
/mnt/sd/usr/lib/libgnutlsxx.so
/mnt/sd/usr/lib/libhistory.so.7.0
/mnt/sd/usr/lib/libSDL2_image-2.0.so.0.2.3
/mnt/sd/usr/lib/liblzo2.so.2
/mnt/sd/usr/lib/libjhead.so
/mnt/sd/usr/lib/libip6tc.so.0
/mnt/sd/usr/lib/libexpat.so.1.6.2
/mnt/sd/usr/lib/libasound.so.2
/mnt/sd/usr/lib/libbacktrace.so
/mnt/sd/usr/lib/libtasn1.so.6
/mnt/sd/usr/lib/libstagefright.so
/mnt/sd/usr/lib/libip6tc.so.0.1.0
/mnt/sd/usr/lib/libmedialistener.so
/mnt/sd/usr/lib/libstateMachineBinder.so
/mnt/sd/usr/lib/libext2fs.so
/mnt/sd/usr/lib/libhogweed.so.4
/mnt/sd/usr/lib/libgthread-2.0.so.0.5000.2
/mnt/sd/usr/lib/libaudiopolicyservice.so
/mnt/sd/usr/lib/libbinder.so
/mnt/sd/usr/lib/libip4tc.so.0
/mnt/sd/usr/lib/libncurses.so
/mnt/sd/usr/lib/libgnutls-openssl.so.27
/mnt/sd/usr/lib/libblkid.so
/mnt/sd/usr/lib/libip4tc.so
/mnt/sd/usr/lib/libasound.so.2.0.0
/mnt/sd/usr/lib/libuuid.so
/mnt/sd/usr/lib/libutils.so
/mnt/sd/usr/lib/libssl.so.1.0.0
/mnt/sd/usr/lib/libjzipu.so
/mnt/sd/usr/lib/libmp4v2.so
/mnt/sd/usr/lib/libi2c.so.0
/mnt/sd/usr/lib/libstagefright_hard_x264hwenc.so
/mnt/sd/usr/lib/libdlog.so
/mnt/sd/usr/lib/libgthread-2.0.so.0
/mnt/sd/usr/lib/libswscale.so
/mnt/sd/usr/lib/libwifi.so
/mnt/sd/usr/lib/libSDL2_image.so
/mnt/sd/usr/lib/libgio-2.0.so
/mnt/sd/usr/lib/libcrypto.so
/mnt/sd/usr/lib/libpcre.so
/mnt/sd/usr/lib/hw
/mnt/sd/usr/lib/hw/audio_policy.default.so
/mnt/sd/usr/lib/hw/camera.default.so
/mnt/sd/usr/lib/hw/audio.primary.default.so
/mnt/sd/usr/lib/hw/local_time.default.so
/mnt/sd/usr/lib/libfreetype.so
/mnt/sd/usr/lib/libmenu.so
/mnt/sd/usr/lib/libswscale.so.5
/mnt/sd/usr/lib/libnl-3.so
/mnt/sd/usr/lib/libFFTEm.so
/mnt/sd/usr/lib/libaudioflinger.so
/mnt/sd/usr/lib/libxtables.so
/mnt/sd/usr/lib/libaudiohelper.so
/mnt/sd/usr/lib/libgnutls.so.30
/mnt/sd/usr/lib/libdbus-1.so
/mnt/sd/usr/lib/libjpeg.so.8
/mnt/sd/usr/lib/libglib-2.0.so
/mnt/sd/usr/lib/libreadline.so.7
/mnt/sd/usr/lib/libexif.so
/mnt/sd/usr/lib/libgio-2.0.so.0
/mnt/sd/usr/lib/libsonivox.so
/mnt/sd/usr/lib/libgmodule-2.0.so.0.5000.2
/mnt/sd/usr/lib/libe2p.so.2
/mnt/sd/usr/lib/libssl.so
/mnt/sd/usr/lib/libusbhost.so
/mnt/sd/usr/lib/libaudioresampler.so
/mnt/sd/usr/lib/libEGL.so
/mnt/sd/usr/lib/libstdc++.so.6
/mnt/sd/usr/lib/libss.so.2.0
/mnt/sd/usr/lib/libhardware.so
/mnt/sd/usr/lib/libSDL2-2.0.so.0.12.0
/mnt/sd/usr/lib/libhistory.so
/mnt/sd/usr/lib/libfreetype.so.6
/mnt/sd/usr/lib/libffi.so.6
/mnt/sd/usr/lib/libsqlite3.so
/mnt/sd/usr/lib/libmount.so
/mnt/sd/usr/lib/libhogweed.so
/mnt/sd/usr/lib/libstagefright_hard_vlume.so
/mnt/sd/usr/lib/libnl-nf-3.so.200
/mnt/sd/usr/lib/libopenobex.so.1
/mnt/sd/usr/lib/libhardware_legacy.so
/mnt/sd/usr/lib/libopenobex.so
/mnt/sd/usr/lib/libcheck.so.0.0.0
/mnt/sd/usr/lib/libavutil.so
/mnt/sd/usr/lib/libgmodule-2.0.so.0
/mnt/sd/usr/lib/libcrypto.so.1.0.0
/mnt/sd/usr/lib/cmake
/mnt/sd/usr/lib/cmake/SDL2
/mnt/sd/usr/lib/cmake/SDL2/sdl2-config-version.cmake
/mnt/sd/usr/lib/cmake/SDL2/sdl2-config.cmake
/mnt/sd/usr/lib/libgmp.so.10.3.2
/mnt/sd/usr/lib/libSDL2.so
/mnt/sd/usr/lib/libmp4v2.so.2
/mnt/sd/usr/lib/libtasn1.so.6.5.5
/mnt/sd/usr/lib/libnl-idiag-3.so.200.22.0
/mnt/sd/usr/lib/libstagefrighthw.so
/mnt/sd/usr/lib/libexpat.so
/mnt/sd/usr/lib/libpanel.so.5.9
/mnt/sd/usr/lib/libcamera_client.so
/mnt/sd/usr/lib/libfreetype.so.6.3.20
/mnt/sd/usr/lib/libmedialogservice.so
/mnt/sd/usr/lib/libmenu.so.5
/mnt/sd/usr/lib/libturbojpeg.so.0
/mnt/sd/usr/lib/libcom_err.so
/mnt/sd/usr/lib/libnl-3.so.200
/mnt/sd/usr/lib/libmenu.so.5.9
/mnt/sd/usr/lib/libjpeg.so.8.1.2
/mnt/sd/usr/lib/libcutils.so
/mnt/sd/usr/lib/libinfiray2_hard_x264hwenc.so
/mnt/sd/usr/lib/libnl-idiag-3.so
/mnt/sd/usr/lib/libwifiservice.so
/mnt/sd/usr/lib/libstagefright_soft_amrnbenc.so
/mnt/sd/usr/lib/libgnutls.so.30.14.11
/mnt/sd/usr/lib/libjzhal.so
/mnt/sd/usr/lib/libss.so
/mnt/sd/usr/lib/libaudioroute.so
/mnt/sd/usr/lib/libui.so
/mnt/sd/usr/lib/libnl-nf-3.so
/mnt/sd/usr/lib/libnl-genl-3.so
/mnt/sd/usr/lib/libpcrecpp.so.0.0.1
/mnt/sd/usr/lib/libmediaListenerBinder.so
/mnt/sd/usr/lib/libunwind.so
/mnt/sd/usr/lib/libmtpservice.so
/mnt/sd/usr/lib/libjzhal.so.m200.3.4.0
/mnt/sd/usr/lib/libturbojpeg.so.0.1.0
/mnt/sd/usr/lib/libnl-xfrm-3.so.200
/mnt/sd/usr/lib/libreadline.so.7.0
/mnt/sd/usr/lib/libncurses.so.5
/mnt/sd/usr/lib/libicuuc.so
/mnt/sd/usr/lib/libpng.so.3
/mnt/sd/usr/lib/libh264stream-test.so
/mnt/sd/usr/lib/libi2c.so.0.1.0
/mnt/sd/usr/lib/libcheck.so
/mnt/sd/usr/lib/libncurses.so.5.9
/mnt/sd/usr/lib/libOMX_Core.so
/mnt/sd/usr/lib/libstagefright_rtsp.so
/mnt/sd/usr/lib/libtinyalsa.so.1
/mnt/sd/usr/lib/libcurses.so
/mnt/sd/usr/lib/libatomic.so.1.1.0
/mnt/sd/usr/lib/libiptc.so.0
/mnt/sd/usr/lib/directfb-1.7-7
/mnt/sd/usr/lib/directfb-1.7-7/gfxdrivers
/mnt/sd/usr/lib/directfb-1.7-7/gfxdrivers/libdirectfb_gal.so
/mnt/sd/usr/lib/libSDL2_ttf-2.0.so.0.14.1
/mnt/sd/usr/lib/libaudioutils.so
/mnt/sd/usr/lib/libstagefright_enc_common.so
/mnt/sd/usr/lib/libCameraMark.so
/mnt/sd/usr/lib/libVDK.so
/mnt/sd/usr/lib/libnl-genl-3.so.200
/mnt/sd/usr/lib/libform.so.5.9
/mnt/sd/usr/lib/libip4tc.so.0.1.0
/mnt/sd/usr/lib/libgnutlsxx.so.28.1.0
/mnt/sd/usr/lib/libcamera_metadata.so
/mnt/sd/usr/lib/libavutil.so.56
/mnt/sd/usr/lib/libvorbisidec.so
/mnt/sd/usr/lib/libmedialistenerservice.so
/mnt/sd/usr/lib/libgnutls.so
/mnt/sd/usr/lib/libspeexresampler.so
/mnt/sd/usr/lib/libgmp.so.10
/mnt/sd/usr/lib/libicui18n.so
/mnt/sd/usr/lib/libstateMachineService.so
/mnt/sd/usr/lib/libnl-3.so.200.22.0
/mnt/sd/usr/lib/libSDL2_ttf.so
/mnt/sd/usr/lib/libssl.so.1.1
/mnt/sd/usr/lib/libasound.so
/mnt/sd/usr/lib/libeffects.so
/mnt/sd/usr/lib/libz.so
/mnt/sd/usr/lib/liblzo2.so.2.0.0
/mnt/sd/usr/lib/libgobject-2.0.so
/mnt/sd/usr/lib/libdbus-1.so.3
/mnt/sd/usr/lib/libGLESv2.so
/mnt/sd/usr/lib/libresourcemanagerservice.so
/mnt/sd/usr/lib/pkgconfig
/mnt/sd/usr/lib/pkgconfig/sdl2.pc
/mnt/sd/usr/lib/libbase.so
/mnt/sd/usr/lib/libiptc.so.0.0.0
/mnt/sd/usr/lib/libcom_err.so.2
/mnt/sd/usr/lib/libnl-route-3.so.200
/mnt/sd/usr/lib/libiptc.so
/mnt/sd/usr/lib/libstagefright_avc_common.so
/mnt/sd/usr/lib/libbluetooth.so.3.13.0
/mnt/sd/usr/lib/libgnutls-openssl.so
/mnt/sd/usr/lib/libmediaplayerservice.so
/mnt/sd/usr/lib/libffi.so
/mnt/sd/usr/lib/libntfs-3g.so.87.0.0
/mnt/sd/usr/lib/libstagefright_soft_mp3dec.so
/mnt/sd/usr/lib/libwifiBinder.so
/mnt/sd/usr/lib/libmedia.so
/mnt/sd/usr/lib/libstateMachine.so
/mnt/sd/usr/lib/libcameraservice.so
/mnt/sd/usr/lib/terminfo
/mnt/sd/usr/lib/libext2fs.so.2.4
/mnt/sd/usr/lib/libnl-idiag-3.so.200
/mnt/sd/usr/lib/libaudiospdif.so
/mnt/sd/usr/lib/libopus.so
/mnt/sd/usr/lib/xtables
/mnt/sd/usr/lib/xtables/libxt_physdev.so
/mnt/sd/usr/lib/xtables/libip6t_REDIRECT.so
/mnt/sd/usr/lib/xtables/libip6t_mh.so
/mnt/sd/usr/lib/xtables/libip6t_SNAT.so
/mnt/sd/usr/lib/xtables/libxt_state.so
/mnt/sd/usr/lib/xtables/libebt_802_3.so
/mnt/sd/usr/lib/xtables/libxt_tcpmss.so
/mnt/sd/usr/lib/xtables/libxt_tos.so
/mnt/sd/usr/lib/xtables/libip6t_REJECT.so
/mnt/sd/usr/lib/xtables/libxt_set.so
/mnt/sd/usr/lib/xtables/libxt_hashlimit.so
/mnt/sd/usr/lib/xtables/libxt_dccp.so
/mnt/sd/usr/lib/xtables/libxt_dscp.so
/mnt/sd/usr/lib/xtables/libxt_AUDIT.so
/mnt/sd/usr/lib/xtables/libip6t_ah.so
/mnt/sd/usr/lib/xtables/libxt_udp.so
/mnt/sd/usr/lib/xtables/libxt_ipcomp.so
/mnt/sd/usr/lib/xtables/libxt_TCPOPTSTRIP.so
/mnt/sd/usr/lib/xtables/libipt_CLUSTERIP.so
/mnt/sd/usr/lib/xtables/libxt_statistic.so
/mnt/sd/usr/lib/xtables/libxt_rpfilter.so
/mnt/sd/usr/lib/xtables/libxt_SECMARK.so
/mnt/sd/usr/lib/xtables/libxt_u32.so
/mnt/sd/usr/lib/xtables/libipt_ECN.so
/mnt/sd/usr/lib/xtables/libip6t_DNPT.so
/mnt/sd/usr/lib/xtables/libxt_pkttype.so
/mnt/sd/usr/lib/xtables/libebt_mark_m.so
/mnt/sd/usr/lib/xtables/libipt_icmp.so
/mnt/sd/usr/lib/xtables/libip6t_MASQUERADE.so
/mnt/sd/usr/lib/xtables/libxt_TCPMSS.so
/mnt/sd/usr/lib/xtables/libxt_NFLOG.so
/mnt/sd/usr/lib/xtables/libxt_connmark.so
/mnt/sd/usr/lib/xtables/libipt_realm.so
/mnt/sd/usr/lib/xtables/libxt_NOTRACK.so
/mnt/sd/usr/lib/xtables/libipt_REJECT.so
/mnt/sd/usr/lib/xtables/libxt_cpu.so
/mnt/sd/usr/lib/xtables/libxt_quota.so
/mnt/sd/usr/lib/xtables/libxt_HMARK.so
/mnt/sd/usr/lib/xtables/libip6t_ipv6header.so
/mnt/sd/usr/lib/xtables/libxt_string.so
/mnt/sd/usr/lib/xtables/libxt_CONNMARK.so
/mnt/sd/usr/lib/xtables/libip6t_rt.so
/mnt/sd/usr/lib/xtables/libxt_CHECKSUM.so
/mnt/sd/usr/lib/xtables/libxt_DSCP.so
/mnt/sd/usr/lib/xtables/libxt_devgroup.so
/mnt/sd/usr/lib/xtables/libip6t_frag.so
/mnt/sd/usr/lib/xtables/libip6t_LOG.so
/mnt/sd/usr/lib/xtables/libxt_limit.so
/mnt/sd/usr/lib/xtables/libxt_SET.so
/mnt/sd/usr/lib/xtables/libxt_CT.so
/mnt/sd/usr/lib/xtables/libxt_owner.so
/mnt/sd/usr/lib/xtables/libxt_RATEEST.so
/mnt/sd/usr/lib/xtables/libipt_SNAT.so
/mnt/sd/usr/lib/xtables/libxt_CONNSECMARK.so
/mnt/sd/usr/lib/xtables/libxt_NFQUEUE.so
/mnt/sd/usr/lib/xtables/libxt_standard.so
/mnt/sd/usr/lib/xtables/libxt_addrtype.so
/mnt/sd/usr/lib/xtables/libipt_ULOG.so
/mnt/sd/usr/lib/xtables/libxt_iprange.so
/mnt/sd/usr/lib/xtables/libxt_policy.so
/mnt/sd/usr/lib/xtables/libipt_TTL.so
/mnt/sd/usr/lib/xtables/libipt_REDIRECT.so
/mnt/sd/usr/lib/xtables/libxt_connbytes.so
/mnt/sd/usr/lib/xtables/libxt_time.so
/mnt/sd/usr/lib/xtables/libxt_MARK.so
/mnt/sd/usr/lib/xtables/libip6t_hbh.so
/mnt/sd/usr/lib/xtables/libxt_TOS.so
/mnt/sd/usr/lib/xtables/libipt_ah.so
/mnt/sd/usr/lib/xtables/libxt_conntrack.so
/mnt/sd/usr/lib/xtables/libxt_TEE.so
/mnt/sd/usr/lib/xtables/libip6t_HL.so
/mnt/sd/usr/lib/xtables/libxt_LED.so
/mnt/sd/usr/lib/xtables/libxt_tcp.so
/mnt/sd/usr/lib/xtables/libxt_mark.so
/mnt/sd/usr/lib/xtables/libxt_nfacct.so
/mnt/sd/usr/lib/xtables/libxt_bpf.so
/mnt/sd/usr/lib/xtables/libxt_mac.so
/mnt/sd/usr/lib/xtables/libxt_ecn.so
/mnt/sd/usr/lib/xtables/libxt_helper.so
/mnt/sd/usr/lib/xtables/libxt_mangle.so
/mnt/sd/usr/lib/xtables/libxt_osf.so
/mnt/sd/usr/lib/xtables/libipt_DNAT.so
/mnt/sd/usr/lib/xtables/libxt_IDLETIMER.so
/mnt/sd/usr/lib/xtables/libip6t_dst.so
/mnt/sd/usr/lib/xtables/libxt_cgroup.so
/mnt/sd/usr/lib/xtables/libipt_NETMAP.so
/mnt/sd/usr/lib/xtables/libipt_MASQUERADE.so
/mnt/sd/usr/lib/xtables/libxt_ipvs.so
/mnt/sd/usr/lib/xtables/libxt_CLASSIFY.so
/mnt/sd/usr/lib/xtables/libxt_connlimit.so
/mnt/sd/usr/lib/xtables/libipt_LOG.so
/mnt/sd/usr/lib/xtables/libxt_recent.so
/mnt/sd/usr/lib/xtables/libebt_log.so
/mnt/sd/usr/lib/xtables/libxt_multiport.so
/mnt/sd/usr/lib/xtables/libxt_rateest.so
/mnt/sd/usr/lib/xtables/libxt_sctp.so
/mnt/sd/usr/lib/xtables/libxt_TRACE.so
/mnt/sd/usr/lib/xtables/libip6t_hl.so
/mnt/sd/usr/lib/xtables/libxt_SYNPROXY.so
/mnt/sd/usr/lib/xtables/libipt_ttl.so
/mnt/sd/usr/lib/xtables/libxt_length.so
/mnt/sd/usr/lib/xtables/libxt_socket.so
/mnt/sd/usr/lib/xtables/libxt_cluster.so
/mnt/sd/usr/lib/xtables/libip6t_SNPT.so
/mnt/sd/usr/lib/xtables/libxt_TPROXY.so
/mnt/sd/usr/lib/xtables/libxt_comment.so
/mnt/sd/usr/lib/xtables/libip6t_eui64.so
/mnt/sd/usr/lib/xtables/libip6t_NETMAP.so
/mnt/sd/usr/lib/xtables/libip6t_DNAT.so
/mnt/sd/usr/lib/xtables/libebt_ip.so
/mnt/sd/usr/lib/xtables/libxt_esp.so
/mnt/sd/usr/lib/xtables/libip6t_icmp6.so
/mnt/sd/usr/lib/libnl.so
/mnt/sd/usr/share
/mnt/sd/usr/share/udhcpc
/mnt/sd/usr/share/udhcpc/default.script
/mnt/sd/usr/share/alsa
/mnt/sd/usr/share/alsa/alsa.conf
/mnt/sd/usr/share/alsa/topology
/mnt/sd/usr/share/alsa/topology/sklrt286
/mnt/sd/usr/share/alsa/topology/sklrt286/skl_i2s.conf
/mnt/sd/usr/share/alsa/topology/bxtrt298
/mnt/sd/usr/share/alsa/topology/bxtrt298/bxt_i2s.conf
/mnt/sd/usr/share/alsa/topology/broadwell
/mnt/sd/usr/share/alsa/topology/broadwell/broadwell.conf
/mnt/sd/usr/share/alsa/sndo-mixer.alisp
/mnt/sd/usr/share/alsa/ucm
/mnt/sd/usr/share/alsa/ucm/VEYRON-I2S
/mnt/sd/usr/share/alsa/ucm/VEYRON-I2S/HiFi.conf
/mnt/sd/usr/share/alsa/ucm/VEYRON-I2S/VEYRON-I2S.conf
/mnt/sd/usr/share/alsa/ucm/PandaBoardES
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/voiceCall
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/FMAnalog
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/hifi
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/voice
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/hifiLP
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/PandaBoardES.conf
/mnt/sd/usr/share/alsa/ucm/PandaBoardES/record
/mnt/sd/usr/share/alsa/ucm/chtrt5645
/mnt/sd/usr/share/alsa/ucm/chtrt5645/HiFi.conf
/mnt/sd/usr/share/alsa/ucm/chtrt5645/chtrt5645.conf
/mnt/sd/usr/share/alsa/ucm/GoogleNyan
/mnt/sd/usr/share/alsa/ucm/GoogleNyan/HiFi.conf
/mnt/sd/usr/share/alsa/ucm/GoogleNyan/GoogleNyan.conf
/mnt/sd/usr/share/alsa/ucm/skylake-rt286
/mnt/sd/usr/share/alsa/ucm/skylake-rt286/Hdmi2
/mnt/sd/usr/share/alsa/ucm/skylake-rt286/Hdmi1
/mnt/sd/usr/share/alsa/ucm/skylake-rt286/HiFi
/mnt/sd/usr/share/alsa/ucm/skylake-rt286/skylake-rt286.conf
/mnt/sd/usr/share/alsa/ucm/tegraalc5632
/mnt/sd/usr/share/alsa/ucm/tegraalc5632/tegraalc5632.conf
/mnt/sd/usr/share/alsa/ucm/SDP4430
/mnt/sd/usr/share/alsa/ucm/SDP4430/voiceCall
/mnt/sd/usr/share/alsa/ucm/SDP4430/FMAnalog
/mnt/sd/usr/share/alsa/ucm/SDP4430/SDP4430.conf
/mnt/sd/usr/share/alsa/ucm/SDP4430/hifi
/mnt/sd/usr/share/alsa/ucm/SDP4430/voice
/mnt/sd/usr/share/alsa/ucm/SDP4430/hifiLP
/mnt/sd/usr/share/alsa/ucm/SDP4430/record
/mnt/sd/usr/share/alsa/ucm/DB410c
/mnt/sd/usr/share/alsa/ucm/DB410c/HDMI
/mnt/sd/usr/share/alsa/ucm/DB410c/DB410c.conf
/mnt/sd/usr/share/alsa/ucm/DB410c/HiFi
/mnt/sd/usr/share/alsa/ucm/PAZ00
/mnt/sd/usr/share/alsa/ucm/PAZ00/HiFi.conf
/mnt/sd/usr/share/alsa/ucm/PAZ00/Record.conf
/mnt/sd/usr/share/alsa/ucm/PAZ00/PAZ00.conf
/mnt/sd/usr/share/alsa/ucm/PandaBoard
/mnt/sd/usr/share/alsa/ucm/PandaBoard/voiceCall
/mnt/sd/usr/share/alsa/ucm/PandaBoard/FMAnalog
/mnt/sd/usr/share/alsa/ucm/PandaBoard/PandaBoard.conf
/mnt/sd/usr/share/alsa/ucm/PandaBoard/hifi
/mnt/sd/usr/share/alsa/ucm/PandaBoard/voice
/mnt/sd/usr/share/alsa/ucm/PandaBoard/hifiLP
/mnt/sd/usr/share/alsa/ucm/PandaBoard/record
/mnt/sd/usr/share/alsa/ucm/broadwell-rt286
/mnt/sd/usr/share/alsa/ucm/broadwell-rt286/broadwell-rt286.conf
/mnt/sd/usr/share/alsa/ucm/broadwell-rt286/HiFi
/mnt/sd/usr/share/alsa/ucm/DAISY-I2S
/mnt/sd/usr/share/alsa/ucm/DAISY-I2S/HiFi.conf
/mnt/sd/usr/share/alsa/ucm/DAISY-I2S/DAISY-I2S.conf
/mnt/sd/usr/share/alsa/alsa.conf.d
/mnt/sd/usr/share/alsa/alsa.conf.d/README
/mnt/sd/usr/share/alsa/pcm
/mnt/sd/usr/share/alsa/pcm/front.conf
/mnt/sd/usr/share/alsa/pcm/hdmi.conf
/mnt/sd/usr/share/alsa/pcm/surround40.conf
/mnt/sd/usr/share/alsa/pcm/iec958.conf
/mnt/sd/usr/share/alsa/pcm/center_lfe.conf
/mnt/sd/usr/share/alsa/pcm/dmix.conf
/mnt/sd/usr/share/alsa/pcm/rear.conf
/mnt/sd/usr/share/alsa/pcm/modem.conf
/mnt/sd/usr/share/alsa/pcm/surround41.conf
/mnt/sd/usr/share/alsa/pcm/surround51.conf
/mnt/sd/usr/share/alsa/pcm/surround71.conf
/mnt/sd/usr/share/alsa/pcm/side.conf
/mnt/sd/usr/share/alsa/pcm/dsnoop.conf
/mnt/sd/usr/share/alsa/pcm/surround50.conf
/mnt/sd/usr/share/alsa/pcm/default.conf
/mnt/sd/usr/share/alsa/pcm/surround21.conf
/mnt/sd/usr/share/alsa/pcm/dpl.conf
/mnt/sd/usr/share/alsa/cards
/mnt/sd/usr/share/alsa/cards/CMI8338-SWIEC.conf
/mnt/sd/usr/share/alsa/cards/PMacToonie.conf
/mnt/sd/usr/share/alsa/cards/CMI8738-MC6.conf
/mnt/sd/usr/share/alsa/cards/VIA686A.conf
/mnt/sd/usr/share/alsa/cards/VXPocket.conf
/mnt/sd/usr/share/alsa/cards/ENS1371.conf
/mnt/sd/usr/share/alsa/cards/aliases.conf
/mnt/sd/usr/share/alsa/cards/AACI.conf
/mnt/sd/usr/share/alsa/cards/CMI8738-MC8.conf
/mnt/sd/usr/share/alsa/cards/RME9652.conf
/mnt/sd/usr/share/alsa/cards/CA0106.conf
/mnt/sd/usr/share/alsa/cards/TRID4DWAVENX.conf
/mnt/sd/usr/share/alsa/cards/SI7018.conf
/mnt/sd/usr/share/alsa/cards/NFORCE.conf
/mnt/sd/usr/share/alsa/cards/YMF744.conf
/mnt/sd/usr/share/alsa/cards/Aureon51.conf
/mnt/sd/usr/share/alsa/cards/AU8810.conf
/mnt/sd/usr/share/alsa/cards/RME9636.conf
/mnt/sd/usr/share/alsa/cards/SI7018
/mnt/sd/usr/share/alsa/cards/SI7018/sndop-mixer.alisp
/mnt/sd/usr/share/alsa/cards/SI7018/sndoc-mixer.alisp
/mnt/sd/usr/share/alsa/cards/Audigy.conf
/mnt/sd/usr/share/alsa/cards/USB-Audio.conf
/mnt/sd/usr/share/alsa/cards/ICE1712.conf
/mnt/sd/usr/share/alsa/cards/CS46xx.conf
/mnt/sd/usr/share/alsa/cards/ATIIXP.conf
/mnt/sd/usr/share/alsa/cards/ATIIXP-SPDMA.conf
/mnt/sd/usr/share/alsa/cards/ENS1370.conf
/mnt/sd/usr/share/alsa/cards/ICE1724.conf
/mnt/sd/usr/share/alsa/cards/EMU10K1.conf
/mnt/sd/usr/share/alsa/cards/Loopback.conf
/mnt/sd/usr/share/alsa/cards/ICH-MODEM.conf
/mnt/sd/usr/share/alsa/cards/VX222.conf
/mnt/sd/usr/share/alsa/cards/FWSpeakers.conf
/mnt/sd/usr/share/alsa/cards/VIA8233A.conf
/mnt/sd/usr/share/alsa/cards/GUS.conf
/mnt/sd/usr/share/alsa/cards/CMI8788.conf
/mnt/sd/usr/share/alsa/cards/Aureon71.conf
/mnt/sd/usr/share/alsa/cards/AU8820.conf
/mnt/sd/usr/share/alsa/cards/FM801.conf
/mnt/sd/usr/share/alsa/cards/VIA8233.conf
/mnt/sd/usr/share/alsa/cards/PMac.conf
/mnt/sd/usr/share/alsa/cards/Echo_Echo3G.conf
/mnt/sd/usr/share/alsa/cards/FireWave.conf
/mnt/sd/usr/share/alsa/cards/AU8830.conf
/mnt/sd/usr/share/alsa/cards/Audigy2.conf
/mnt/sd/usr/share/alsa/cards/ICH.conf
/mnt/sd/usr/share/alsa/cards/SB-XFi.conf
/mnt/sd/usr/share/alsa/cards/PS3.conf
/mnt/sd/usr/share/alsa/cards/ES1968.conf
/mnt/sd/usr/share/alsa/cards/VXPocket440.conf
/mnt/sd/usr/share/alsa/cards/ATIIXP-MODEM.conf
/mnt/sd/usr/share/alsa/cards/Maestro3.conf
/mnt/sd/usr/share/alsa/cards/ICH4.conf
/mnt/sd/usr/share/alsa/cards/aliases.alisp
/mnt/sd/usr/share/alsa/cards/EMU10K1X.conf
/mnt/sd/usr/share/alsa/cards/CMI8338.conf
/mnt/sd/usr/share/alsa/cards/VIA8237.conf
/mnt/sd/usr/share/alsa/cards/PC-Speaker.conf
/mnt/sd/usr/share/alsa/cards/HDA-Intel.conf
/mnt/sd/usr/share/gettext
/mnt/sd/usr/share/gettext/its
/mnt/sd/usr/share/gettext/its/gschema.its
/mnt/sd/usr/share/gettext/its/gschema.loc
/mnt/sd/usr/share/bash-completion
/mnt/sd/usr/share/bash-completion/completions
/mnt/sd/usr/share/bash-completion/completions/gapplication
/mnt/sd/usr/share/bash-completion/completions/gresource
/mnt/sd/usr/share/bash-completion/completions/gsettings
/mnt/sd/usr/share/bash-completion/completions/uuidgen
/mnt/sd/usr/share/bash-completion/completions/gdbus
/mnt/sd/usr/share/aclocal
/mnt/sd/usr/share/aclocal/gsettings.m4
/mnt/sd/usr/share/aclocal/glib-gettext.m4
/mnt/sd/usr/share/aclocal/glib-2.0.m4
/mnt/sd/usr/share/gdb
/mnt/sd/usr/share/gdb/auto-load
/mnt/sd/usr/share/gdb/auto-load/libglib-2.0.so.0.4000.0-gdb.py
/mnt/sd/usr/share/glib-2.0
/mnt/sd/usr/share/glib-2.0/gdb
/mnt/sd/usr/share/glib-2.0/gdb/glib_gdb.py
/mnt/sd/usr/share/glib-2.0/gdb/gobject_gdb.py
/mnt/sd/usr/share/glib-2.0/gdb/glib.py
/mnt/sd/usr/share/images
/mnt/sd/usr/share/images/PS-D效果图
/mnt/sd/usr/share/images/PS-D效果图/1开关机
/mnt/sd/usr/share/images/PS-D效果图/1开关机/图片36.png
/mnt/sd/usr/share/images/PS-D效果图/1开关机/图片1.png
/mnt/sd/usr/share/images/PS-D效果图/0全部
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片30.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片11.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片12.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片16.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片5.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片4.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片15.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片27.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片19.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片10.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片17.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片13.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片7.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片6.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片33.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片23.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片34.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片29.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片36.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片8.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片22.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片26.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片1.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片9.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片24.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片21.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片2.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片18.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片31.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片20.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片32.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片25.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片3.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片14.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片28.png
/mnt/sd/usr/share/images/PS-D效果图/0全部/图片35.png
/mnt/sd/usr/share/images/PS-D效果图/2主界面
/mnt/sd/usr/share/images/PS-D效果图/2主界面/图片4.png
/mnt/sd/usr/share/images/PS-D效果图/2主界面/图片2.png
/mnt/sd/usr/share/images/PS-D效果图/2主界面/图片3.png
/mnt/sd/usr/share/images/PS-D效果图/菜单
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择6文件管理
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择6文件管理/图片21.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择6文件管理/图片20.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/图片5.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择7快速设置
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片23.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片22.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片26.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片24.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片25.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/图片19.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择5目标距离
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择5目标距离/图片18.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择3电子变倍
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择3电子变倍/图片12.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片30.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片27.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片33.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片34.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片29.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片31.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片32.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片28.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片35.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择2红外图像
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片11.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片10.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片7.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片8.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片9.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择1屏幕亮度
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择1屏幕亮度/图片6.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择4分划设置
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片16.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片15.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片17.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片13.png
/mnt/sd/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片14.png
/mnt/sd/usr/share/images/图片11.png
/mnt/sd/usr/share/images/背景.png
/mnt/sd/usr/share/images/beijing3.png
/mnt/sd/usr/share/images/beijing.png
/mnt/sd/usr/share/images/PS-D960_768切图
/mnt/sd/usr/share/images/PS-D960_768切图/常显
/mnt/sd/usr/share/images/PS-D960_768切图/常显/4电池-4.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/4电池-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0顶栏背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/1可见光图像.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/其它
/mnt/sd/usr/share/images/PS-D960_768切图/常显/其它/z西.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/其它/z北.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/其它/z南.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/其它/z东.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0顶栏背景-遮罩.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0右栏背景-遮罩.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/1白热.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/1红热.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/1伪彩.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/2电子变倍背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/3录像.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0右栏背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/负号.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/点.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/度.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/4电池.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/5俯仰角三角.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0目标距离.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-2.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-4.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-8.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-0.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-6.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/电子变倍数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/4电池-2.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/4电池-5.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/3拍照.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/1红外图像.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0底栏背景 右.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/分划/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/数码管数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/1黑热.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/5方位角背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/5方位角三角.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/5俯仰背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/2电子变倍背景 x .png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/4电池-3.png
/mnt/sd/usr/share/images/PS-D960_768切图/常显/0底栏背景 左.png
/mnt/sd/usr/share/images/PS-D960_768切图/中.png
/mnt/sd/usr/share/images/PS-D960_768切图/俄.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/1激光.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/其它
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/其它/1.bmp
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/3进度条-2.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/1电子罗盘.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/3进度条.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/4关.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/3进度条-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/2异常.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/1省略号.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/1模组.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/1电池.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/4关-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/1背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/开关机/2正常.png
/mnt/sd/usr/share/images/PS-D960_768切图/语言切换.png
/mnt/sd/usr/share/images/PS-D960_768切图/法.png
/mnt/sd/usr/share/images/PS-D960_768切图/英语.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速设置.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/3罗盘.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/ON.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/off.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化/选中.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/3罗盘
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/3罗盘/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/-.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/确定.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/四字框.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/取消.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/两字框.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/8故障
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/8故障/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/8故障.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复/选中.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/0背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/退出.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/菜单.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/Y.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/X.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/cancel.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/确定.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/分划.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/add.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/射表.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/选中.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/选择位数.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/射表背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/枪型.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/弹种.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划位置.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1取消.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/2正号.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1确定.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0数值背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0X标尺.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/2分划.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0Y标尺.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1X.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/2负号.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1数值选中.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1Y.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0菜单.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划样式.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划亮度.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/绿色.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/蓝色.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/白色.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/黑色.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/红色.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/背景校正-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/4背景校正.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/1对比度.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/3图像模式.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/2图像增强.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/0背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/0亮度.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/2红外图像/未标题-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/3电子变倍
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/3电子变倍/变倍-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/8高级设置.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-0.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/2图片.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/2视频.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/es.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/fra.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/rus.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/en.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/cn.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/0文件选框.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3下一张.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3下一张-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快进-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/4文件名背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3上一张-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0暂停-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/1播放进度条-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/1播放进度条.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3上一张.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0播放-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快进.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0播放.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快退-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0暂停.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快退.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-4.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/0大背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-3.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/0菜单.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/0视频播放图标.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/6文件管理/2帮助.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/3变倍设置.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/选中.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/8.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/6.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/2.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/9.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/4.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/7.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/5.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/0.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/3.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/确认.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/5目标距离/取消.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/0背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/OFF.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/快速瞄准.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/蓝牙.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/wifi.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/on.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/视频传输.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/OFF-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/分划.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/0背景.png
/mnt/sd/usr/share/images/PS-D960_768切图/菜单/7快速调整/on-1.png
/mnt/sd/usr/share/images/PS-D960_768切图/西.png
/mnt/sd/usr/share/dbus-1
/mnt/sd/usr/share/dbus-1/system-services
/mnt/sd/usr/share/dbus-1/system-services/org.bluez.service
/mnt/sd/usr/share/dbus-1/session.conf
/mnt/sd/usr/share/dbus-1/system.conf
/mnt/sd/usr/share/terminfo
/mnt/sd/usr/share/terminfo/p
/mnt/sd/usr/share/terminfo/p/putty
/mnt/sd/usr/share/terminfo/p/putty-vt100
/mnt/sd/usr/share/terminfo/a
/mnt/sd/usr/share/terminfo/a/ansi
/mnt/sd/usr/share/terminfo/v
/mnt/sd/usr/share/terminfo/v/vt102
/mnt/sd/usr/share/terminfo/v/vt200
/mnt/sd/usr/share/terminfo/v/vt100-putty
/mnt/sd/usr/share/terminfo/v/vt100
/mnt/sd/usr/share/terminfo/v/vt220
/mnt/sd/usr/share/terminfo/s
/mnt/sd/usr/share/terminfo/s/screen
/mnt/sd/usr/share/terminfo/l
/mnt/sd/usr/share/terminfo/l/linux
/mnt/sd/usr/share/terminfo/x
/mnt/sd/usr/share/terminfo/x/xterm-xfree86
/mnt/sd/usr/share/terminfo/x/xterm
/mnt/sd/usr/share/terminfo/x/xterm-color
/mnt/sd/usr/lib32
/mnt/sd/usr/data
/mnt/sd/lib32
/mnt/sd/data
/mnt/sd/data/database
/mnt/sd/data/database/mtp.db
/mnt/sd/var
/mnt/sd/var/run
/mnt/sd/var/tmp
/mnt/sd/var/www
/mnt/sd/var/www/update.html
/mnt/sd/var/www/halmic.html
/mnt/sd/var/www/flashing.html
/mnt/sd/var/www/fail.html
/mnt/sd/var/www/art.html
/mnt/sd/var/www/update.asp
/mnt/sd/var/www/style.css
/mnt/sd/var/www/halfpga.html
/mnt/sd/var/www/404.html
/mnt/sd/var/www/cgi-bin
/mnt/sd/var/www/cgi-bin/setdevicecfg.cgi
/mnt/sd/var/www/cgi-bin/setpasswd
/mnt/sd/var/www/cgi-bin/pushfile.cgi
/mnt/sd/var/www/cgi-bin/cgictest.cgi
/mnt/sd/var/www/cgi-bin/trio_tmr
/mnt/sd/var/www/cgi-bin/productInfo2
/mnt/sd/var/www/cgi-bin/setssid
/mnt/sd/var/www/cgi-bin/version
/mnt/sd/var/www/cgi-bin/getpasswd
/mnt/sd/var/www/cgi-bin/pushfpga.cgi
/mnt/sd/var/www/cgi-bin/getssid
/mnt/sd/var/www/cgi-bin/productVersion
/mnt/sd/var/www/cgi-bin/cgijson.cgi
/mnt/sd/var/www/mac.html
/mnt/sd/var/lock
/mnt/sd/var/lib
/mnt/sd/var/lib/misc
/mnt/sd/var/lib/dbus
/mnt/sd/var/spool
/mnt/sd/var/cache
/mnt/sd/var/log
/mnt/sd/storage
/mnt/sd/sys
/mnt/sd/sys/.gitignore
/mnt/sd/dev
/mnt/sd/dev/console
/mnt/sd/dev/null
/mnt/sd/dev/log
/mnt/.gitignore
/sbin
/sbin/mkswap
/sbin/route
/sbin/iptunnel
/sbin/mkfs.minix
/sbin/mkfs.msdos
/sbin/wifi_connect.sh
/sbin/logread
/sbin/adbserver.sh
/sbin/ipneigh
/sbin/ipaddr
/sbin/mkfs.vfat
/sbin/ifup
/sbin/devmem
/sbin/mkfs.ext2
/sbin/uevent
/sbin/adjtimex
/sbin/raidautorun
/sbin/setconsole
/sbin/mkdosfs
/sbin/adbd
/sbin/makedevs
/sbin/iwgetid
/sbin/iwpriv
/sbin/tunctl
/sbin/klogd
/sbin/swapoff
/sbin/ap_wifi_sta.sh
/sbin/ip
/sbin/findfs
/sbin/fstrim
/sbin/lsmod
/sbin/vconfig
/sbin/ifconfig
/sbin/update
/sbin/blkid
/sbin/getpackage
/sbin/switch_root
/sbin/acpid
/sbin/fsck.minix
/sbin/hwclock
/sbin/iprule
/sbin/mkfs.fat
/sbin/mount.lowntfs-3g
/sbin/init
/sbin/wifi_start
/sbin/swapon
/sbin/loadkmap
/sbin/getty
/sbin/iwconfig
/sbin/depmod
/sbin/fbsplash
/sbin/iproute
/sbin/blockdev
/sbin/zcip
/sbin/pivot_root
/sbin/sysctl
/sbin/arp
/sbin/reboot
/sbin/modprobe
/sbin/ifenslave
/sbin/rmmod
/sbin/bootchartd
/sbin/syslogd
/sbin/iwspy
/sbin/runlevel
/sbin/akiss
/sbin/airkiss
/sbin/watchdog
/sbin/slattach
/sbin/mod_init.sh
/sbin/iplink
/sbin/sulogin
/sbin/mount.ntfs-3g
/sbin/freeramdisk
/sbin/modinfo
/sbin/ifdown
/sbin/nameif
/sbin/losetup
/sbin/hdparm
/sbin/insmod
/sbin/mdev
/sbin/iwlist
/sbin/udhcpc
/sbin/mke2fs
/sbin/fdisk
/sbin/halt
/sbin/wifi_stop.sh
/sbin/start-stop-daemon
/sbin/sta_wifi_ap.sh
/sbin/mod_install_all.sh
/sbin/poweroff
/bin
/bin/dumpkmap
/bin/mv
/bin/setarch
/bin/rm
/bin/rmdir
/bin/catv
/bin/ping6
/bin/egrep
/bin/fatattr
/bin/hostname
/bin/mpstat
/bin/sed
/bin/rev
/bin/gzip
/bin/ln
/bin/chgrp
/bin/ash
/bin/getopt
/bin/bash
/bin/umount
/bin/pwd
/bin/sync
/bin/busybox
/bin/true
/bin/dnsdomainname
/bin/gunzip
/bin/watch
/bin/dmesg
/bin/grep
/bin/date
/bin/conspy
/bin/ping
/bin/base64
/bin/mktemp
/bin/kbd_mode
/bin/zcat
/bin/tar
/bin/ls
/bin/false
/bin/chown
/bin/sh
/bin/nice
/bin/hush
/bin/ps
/bin/stty
/bin/setserial
/bin/cttyhack
/bin/makemime
/bin/more
/bin/fdflush
/bin/stat
/bin/kill
/bin/mt
/bin/pidof
/bin/cat
/bin/scriptreplay
/bin/printenv
/bin/fgrep
/bin/chmod
/bin/reformime
/bin/run-parts
/bin/cpio
/bin/pipe_progress
/bin/dd
/bin/fsync
/bin/vi
/bin/ionice
/bin/su
/bin/df
/bin/mount
/bin/usleep
/bin/mountpoint
/bin/echo
/bin/netstat
/bin/sleep
/bin/touch
/bin/uname
/bin/ipcalc
/bin/mknod
/bin/ed
/bin/login
/bin/linux64
/bin/cp
/bin/mkdir
/bin/iostat
/bin/linux32
/run
/run/udhcpd.pid
/run/syslogd.pid
/run/network
/run/telnetd.pid
/run/klogd.pid
/run/utmp
/run/dbus
/run/dbus/system_bus_socket
/run/dbus/pid
/run/ifstate
/run/blkid
/run/blkid/blkid.tab
/run/readme
/etc
/etc/inputrc
/etc/os-release
/etc/httpd.conf
/etc/nsswitch.conf
/etc/media_codecs_google_video.xml
/etc/inittab
/etc/hosts
/etc/network
/etc/network/if-pre-up.d
/etc/network/if-pre-up.d/wait_iface
/etc/network/interfaces
/etc/ota_res
/etc/ota_res/update.sh
/etc/ota_res/recovery.conf
/etc/ota_res/key
/etc/ota_res/key/key.pub
/etc/ota_res/recovery
/etc/ota_res/update_stage.sh
/etc/ota_res/update_play.sh
/etc/media_codecs.xml
/etc/ld.so.cache
/etc/hostapd.deny
/etc/resolv.conf
/etc/appenv.sh
/etc/mke2fs.conf
/etc/random-seed
/etc/usb
/etc/usb/usb_removing
/etc/usb/usb_inserting
/etc/dmesg_log.sh
/etc/mixer_paths.xml
/etc/fstab
/etc/init.d
/etc/init.d/S40copy_ota_res
/etc/init.d/S50telnet
/etc/init.d/S20parameter
/etc/init.d/S20urandom
/etc/init.d/S40network
/etc/init.d/S22debuggerd
/etc/init.d/S90dlog
/etc/init.d/rcS
/etc/init.d/S10mdev
/etc/init.d/TS90misc
/etc/init.d/S30dbus
/etc/init.d/adb
/etc/init.d/adb/S310adb
/etc/init.d/adb/S440adb
/etc/init.d/S12storage
/etc/init.d/S01logging
/etc/init.d/S91app
/etc/init.d/S21data
/etc/init.d/S90misc
/etc/init.d/S20mntsd
/etc/init.d/S90adb
/etc/init.d/rcK
/etc/bluetooth
/etc/bluetooth/rfcomm.conf
/etc/bluetooth/main.conf
/etc/wpa_supplicant.conf
/etc/media_profiles.xml
/etc/protocols
/etc/services
/etc/superui.cfg
/etc/profile.d
/etc/profile.d/superui.sh
/etc/profile.d/umask.sh
/etc/profile.d/debugger.sh
/etc/group
/etc/passwd
/etc/ssl
/etc/ssl/openssl.cnf
/etc/media_codecs_google_audio.xml
/etc/shadow
/etc/hostapd.conf
/etc/mtab
/etc/udhcpd.conf
/etc/sudoers
/etc/mdev.conf
/etc/profile
/etc/dbus-1
/etc/dbus-1/system.d
/etc/dbus-1/system.d/pulseaudio-system.conf
/etc/dbus-1/system.d/wpa_supplicant.conf
/etc/dbus-1/system.d/bluetooth.conf
/etc/dbus-1/session.conf
/etc/dbus-1/system.conf
/etc/audio_policy.conf
/etc/libnl
/etc/libnl/classid
/etc/libnl/pktloc
/tmp
/tmp/bt_connect_status
/tmp/messages
/tmp/messages.0
/tmp/tombstones
/tmp/tombstones/tombstone_00
/tmp/gatttoolpid
/tmp/cameraindex2
/tmp/cameraindex1
/tmp/cameraindex0
/tmp/hostapd.conf
/tmp/udhcpd.leases
/tmp/rotateflag
/tmp/UIindex2
/tmp/UIindex1
/tmp/UIindex0
/tmp/dbus
/tmp/dbus/machine-id
/tmp/subsys
/tmp/version
/tmp/bt_connect_information
/tmp/bt_information
/tmp/bt_bluez_en
/tmp/bt_ready_flag
/linuxrc
/root
/root/.gitignore
/root/.ash_history
/lib
/lib/libmount.so.1.1.0
/lib/libblkid.so.1
/lib/libuuid.so.1
/lib/libdl.so.2
/lib/libcrypt-2.22.so
/lib/libatomic.so.1
/lib/libblkid.so.1.1.0
/lib/libthread_db-1.0.so
/lib/libnsl.so.1
/lib/libnss_files.so.2
/lib/libnsl-2.22.so
/lib/libm-2.22.so
/lib/libc.so.6
/lib/libgcc_s.so.1
/lib/libm.so.6
/lib/librt-2.22.so
/lib/ld.so.1
/lib/libnss_files-2.22.so
/lib/libuuid.so.1.3.0
/lib/libutil-2.22.so
/lib/libnss_dns.so.2
/lib/libmount.so.1
/lib/firmware
/lib/firmware/BCM4343A1_001.002.009.0025.0059.hcd
/lib/libanl.so.1
/lib/libcrypt.so.1
/lib/bash
/lib/bash/printenv
/lib/bash/rmdir
/lib/bash/print
/lib/bash/uname
/lib/bash/logname
/lib/bash/strftime
/lib/bash/head
/lib/bash/whoami
/lib/bash/ln
/lib/bash/tty
/lib/bash/Makefile.inc
/lib/bash/sleep
/lib/bash/mkdir
/lib/bash/realpath
/lib/bash/setpgid
/lib/bash/pathchk
/lib/bash/sync
/lib/bash/unlink
/lib/bash/id
/lib/bash/dirname
/lib/bash/tee
/lib/bash/mypid
/lib/bash/basename
/lib/bash/finfo
/lib/bash/push
/lib/bash/truefalse
/lib/libc-2.22.so
/lib/libdl-2.22.so
/lib/libanl-2.22.so
/lib/libpthread.so.0
/lib/ld-2.22.so
/lib/libutil.so.1
/lib/libatomic.so.1.1.0
/lib/libresolv.so.2
/lib/libnss_dns-2.22.so
/lib/pkgconfig
/lib/pkgconfig/bash.pc
/lib/libpthread-2.22.so
/lib/libresolv-2.22.so
/lib/libthread_db.so.1
/lib/librt.so.1
/firmware
/firmware/wifimac.txt
/firmware/nv_43438_a1.cal
/firmware/fw_43438_a1.bin
/parameter
/parameter/superui.cfg
/parameter/lost+found
/parameter/product_SN
/parameter/parameter.cfg
/parameter/device.cfg
/parameter/product_PN
/opt
/opt/.gitignore
/lost+found
/usr
/usr/libexec
/usr/libexec/dbus-daemon-launch-helper
/usr/libexec/sudo
/usr/libexec/sudo/libsudo_util.so
/usr/libexec/sudo/sudoers.so
/usr/libexec/sudo/sudo_noexec.so
/usr/libexec/sudo/group_file.so
/usr/libexec/sudo/libsudo_util.so.0.0.0
/usr/libexec/sudo/libsudo_util.so.0
/usr/libexec/sudo/system_group.so
/usr/sbin
/usr/sbin/delgroup
/usr/sbin/svlogd
/usr/sbin/bspatch
/usr/sbin/mkfs.ext4dev
/usr/sbin/ubirsvol
/usr/sbin/ip6tables-restore
/usr/sbin/sendmail
/usr/sbin/chpasswd
/usr/sbin/fbset
/usr/sbin/ubirename
/usr/sbin/popmaildir
/usr/sbin/httpd
/usr/sbin/nbd-client
/usr/sbin/ubiformat
/usr/sbin/ntpd
/usr/sbin/mkfs.ext2
/usr/sbin/bluetoothd
/usr/sbin/lpd
/usr/sbin/hciconfig
/usr/sbin/chroot
/usr/sbin/rdev
/usr/sbin/rtcwake
/usr/sbin/tftpd
/usr/sbin/readahead
/usr/sbin/chat
/usr/sbin/loadfont
/usr/sbin/flash_unlock
/usr/sbin/ubimkvol
/usr/sbin/i2cset
/usr/sbin/ip6tables-save
/usr/sbin/ubiupdatevol
/usr/sbin/killall5
/usr/sbin/nandtest
/usr/sbin/ubicrc32
/usr/sbin/telnetd
/usr/sbin/hciattach
/usr/sbin/ip6tables
/usr/sbin/ether-wake
/usr/sbin/i2cget
/usr/sbin/dnsd
/usr/sbin/bsdiff
/usr/sbin/remove-shell
/usr/sbin/adduser
/usr/sbin/visudo
/usr/sbin/e4crypt
/usr/sbin/wpa_supplicant
/usr/sbin/crond
/usr/sbin/ubirmvol
/usr/sbin/m200_wifi_sta.sh
/usr/sbin/flash_lock
/usr/sbin/ubiattach
/usr/sbin/xtables-multi
/usr/sbin/mtd_debug
/usr/sbin/fakeidentd
/usr/sbin/setfont
/usr/sbin/dhcprelay
/usr/sbin/ubiblock
/usr/sbin/i2cdetect
/usr/sbin/ifplugd
/usr/sbin/flashcp
/usr/sbin/readprofile
/usr/sbin/addgroup
/usr/sbin/mkfs.jffs2
/usr/sbin/mkfs.ext3
/usr/sbin/ubinfo
/usr/sbin/nandwrite
/usr/sbin/ftpd
/usr/sbin/ftl_check
/usr/sbin/recovery
/usr/sbin/iptables-save
/usr/sbin/inetd
/usr/sbin/i2cdump
/usr/sbin/mkfs.ext4
/usr/sbin/arping
/usr/sbin/nanddump
/usr/sbin/deluser
/usr/sbin/add-shell
/usr/sbin/brctl
/usr/sbin/iptables-restore
/usr/sbin/fdformat
/usr/sbin/ubinize
/usr/sbin/flash_erase
/usr/sbin/ubidetach
/usr/sbin/powertop
/usr/sbin/mke2fs
/usr/sbin/rdate
/usr/sbin/setlogcons
/usr/sbin/hostapd
/usr/sbin/iptables
/usr/sbin/udhcpd
/usr/sbin/mtdinfo
/usr/bin
/usr/bin/unxz
/usr/bin/dbus-uuidgen
/usr/bin/dbus-cleanup-sockets
/usr/bin/ircp
/usr/bin/yes
/usr/bin/pcregrep
/usr/bin/reset
/usr/bin/gapplication
/usr/bin/uuencode
/usr/bin/mesg
/usr/bin/expr
/usr/bin/ble
/usr/bin/nsenter
/usr/bin/nohup
/usr/bin/tinycap
/usr/bin/dlog_logger
/usr/bin/sdl2-config
/usr/bin/lspci
/usr/bin/lpq
/usr/bin/update_checking
/usr/bin/obex_test
/usr/bin/dos2unix
/usr/bin/ntfs-3g.secaudit
/usr/bin/groups
/usr/bin/logname
/usr/bin/traceroute6
/usr/bin/stateMachineServer
/usr/bin/resize
/usr/bin/less
/usr/bin/ar
/usr/bin/uptime
/usr/bin/whois
/usr/bin/last
/usr/bin/hostapd_cli
/usr/bin/lzcat
/usr/bin/rfcomm
/usr/bin/screen_out_test
/usr/bin/dbus-test-tool
/usr/bin/man
/usr/bin/install
/usr/bin/strace-log-merge
/usr/bin/pcretest
/usr/bin/tftp
/usr/bin/uniq
/usr/bin/bzip2
/usr/bin/arecord
/usr/bin/ldconfig
/usr/bin/fgconsole
/usr/bin/tinymix
/usr/bin/lowntfs-3g
/usr/bin/setuidgid
/usr/bin/agent
/usr/bin/i2ctransfer
/usr/bin/mkfifo
/usr/bin/pgrep
/usr/bin/svc
/usr/bin/md5sum
/usr/bin/envuidgid
/usr/bin/bluetoothctl
/usr/bin/clear
/usr/bin/chrt
/usr/bin/head
/usr/bin/nmeter
/usr/bin/bt_enable_a0
/usr/bin/du
/usr/bin/ble_server
/usr/bin/sum
/usr/bin/dbus-send
/usr/bin/pkill
/usr/bin/wpa_supplicant
/usr/bin/sort
/usr/bin/whoami
/usr/bin/uudecode
/usr/bin/awk
/usr/bin/cut
/usr/bin/gdbserver
/usr/bin/dpkg
/usr/bin/ttysize
/usr/bin/logger
/usr/bin/dbus-monitor
/usr/bin/ftpput
/usr/bin/sudoreplay
/usr/bin/gresource
/usr/bin/bt_enable
/usr/bin/cryptpw
/usr/bin/deallocvt
/usr/bin/eject
/usr/bin/dpkg-split
/usr/bin/split
/usr/bin/sudo
/usr/bin/RTSPServer
/usr/bin/readlink
/usr/bin/ntfs-3g.probe
/usr/bin/lsusb
/usr/bin/unzip
/usr/bin/udc_mass_storage.sh
/usr/bin/killall
/usr/bin/beep
/usr/bin/dpkg-deb
/usr/bin/script
/usr/bin/nslookup
/usr/bin/dbus-update-activation-environment
/usr/bin/softlimit
/usr/bin/unlzma
/usr/bin/dlogutil
/usr/bin/tty
/usr/bin/runsv
/usr/bin/openvt
/usr/bin/aserver
/usr/bin/dlogctrl
/usr/bin/sha3sum
/usr/bin/who
/usr/bin/wc
/usr/bin/irobex_palm3
/usr/bin/pscan
/usr/bin/hexdump
/usr/bin/uyuv_to_nv12
/usr/bin/env
/usr/bin/vlock
/usr/bin/pmap
/usr/bin/traceroute
/usr/bin/hd
/usr/bin/setsid
/usr/bin/dumpleases
/usr/bin/tcpsvd
/usr/bin/dbus-daemon
/usr/bin/printf
/usr/bin/blkdiscard
/usr/bin/cal
/usr/bin/openssl
/usr/bin/smemcap
/usr/bin/realpath
/usr/bin/envdir
/usr/bin/servicemanager
/usr/bin/tail
/usr/bin/showkey
/usr/bin/wget
/usr/bin/od
/usr/bin/ftpget
/usr/bin/tinyplay
/usr/bin/sv
/usr/bin/aplay
/usr/bin/lpr
/usr/bin/iptables-xml
/usr/bin/gio
/usr/bin/sdptool
/usr/bin/gsettings
/usr/bin/strings
/usr/bin/microcom
/usr/bin/lsof
/usr/bin/TakePicture
/usr/bin/ntfs-3g.usermap
/usr/bin/wpa_cli
/usr/bin/xz
/usr/bin/gdbus
/usr/bin/xargs
/usr/bin/ciptool
/usr/bin/gatttool
/usr/bin/irxfer
/usr/bin/telnet
/usr/bin/find
/usr/bin/time
/usr/bin/tree
/usr/bin/mediaserver
/usr/bin/brcm_patchram_plus
/usr/bin/[
/usr/bin/wifiserver
/usr/bin/rx
/usr/bin/pminstall
/usr/bin/sha512sum
/usr/bin/ntfs-3g
/usr/bin/gio-querymodules
/usr/bin/xzcat
/usr/bin/VideoRecord
/usr/bin/chvt
/usr/bin/dbus-run-session
/usr/bin/sudoedit
/usr/bin/volname
/usr/bin/which
/usr/bin/start-stop-daemon
/usr/bin/tinypcminfo
/usr/bin/sha1sum
/usr/bin/bzcat
/usr/bin/AudioRecord
/usr/bin/test-debugger
/usr/bin/jpegtran
/usr/bin/amixer
/usr/bin/ipcrm
/usr/bin/unshare
/usr/bin/udpsvd
/usr/bin/l2ping
/usr/bin/dbus-launch
/usr/bin/passwd
/usr/bin/runsvdir
/usr/bin/[[
/usr/bin/wrjpgcom
/usr/bin/crontab
/usr/bin/tr
/usr/bin/WifiTest
/usr/bin/unlink
/usr/bin/bt_enable_a1
/usr/bin/rdjpgcom
/usr/bin/strace
/usr/bin/setkeycodes
/usr/bin/cksum
/usr/bin/udhcpc6
/usr/bin/patch
/usr/bin/sqlite3
/usr/bin/udc_mass_storage
/usr/bin/timeout
/usr/bin/MediaPlayer
/usr/bin/cjpeg
/usr/bin/hcitool
/usr/bin/chpst
/usr/bin/fold
/usr/bin/mkpasswd
/usr/bin/hostapd
/usr/bin/hostapd/wlan0
/usr/bin/mtpserver
/usr/bin/pstree
/usr/bin/ipcs
/usr/bin/seq
/usr/bin/debuggerd
/usr/bin/test_recoder
/usr/bin/sha256sum
/usr/bin/dc
/usr/bin/djpeg
/usr/bin/nc
/usr/bin/obex_tcp
/usr/bin/superui
/usr/bin/id
/usr/bin/dirname
/usr/bin/users
/usr/bin/unix2dos
/usr/bin/truncate
/usr/bin/hostid
/usr/bin/test
/usr/bin/pwdx
/usr/bin/fuser
/usr/bin/tee
/usr/bin/cmp
/usr/bin/diff
/usr/bin/basename
/usr/bin/renice
/usr/bin/free
/usr/bin/lzma
/usr/bin/flock
/usr/bin/wall
/usr/bin/top
/usr/etc
/usr/etc/dlog.conf
/usr/etc/virtual_uart.sh
/usr/etc/dlog_logger.conf
/usr/icu
/usr/icu/icudt55l.dat
/usr/lib
/usr/lib/libGAL.so
/usr/lib/libstagefright_hard_alume.so
/usr/lib/libe2p.so.2.3
/usr/lib/libe2p.so
/usr/lib/libnl-xfrm-3.so.200.22.0
/usr/lib/libntfs-3g.so
/usr/lib/libglib-2.0.so.0.5000.2
/usr/lib/libdbus-1.so.3.14.10
/usr/lib/libmp4v2.so.2.0.0
/usr/lib/libntfs-3g.so.87
/usr/lib/libstagefright_soft_amrwbenc.so
/usr/lib/libpcreposix.so.0
/usr/lib/libnettle.so.6.4
/usr/lib/libgmp.so
/usr/lib/libnl-route-3.so.200.22.0
/usr/lib/libpcre.so.1
/usr/lib/libgui.so
/usr/lib/libhogweed.so.4.4
/usr/lib/libsonic.so
/usr/lib/libstagefright_wfd.so
/usr/lib/libtinyalsa.so.1.1.0
/usr/lib/libopenobex.so.1.4.1
/usr/lib/libpng.so
/usr/lib/libstagefright_soft_amrdec.so
/usr/lib/libSDL2_ttf-2.0.so.0
/usr/lib/libgobject-2.0.so.0
/usr/lib/libpcreposix.so
/usr/lib/libtasn1.so
/usr/lib/liblzo2.so
/usr/lib/libcommon_time_client.so
/usr/lib/libatomic.so.1
/usr/lib/libVIVANTE.so
/usr/lib/libxtables.so.12
/usr/lib/libgthread-2.0.so
/usr/lib/libmtp.so
/usr/lib/libstagefright_amrnb_common.so
/usr/lib/libz.so.1.2.8
/usr/lib/libgobject-2.0.so.0.5000.2
/usr/lib/libstdc++.so.6.0.21
/usr/lib/libss.so.2
/usr/lib/libpcrecpp.so.0
/usr/lib/libnl-route-3.so
/usr/lib/libstagefright_soft_aacenc.so
/usr/lib/libnettle.so.6
/usr/lib/libwpa_client.so
/usr/lib/libjpeg.so.8.0.2
/usr/lib/libhistory.so.7
/usr/lib/libgnutls-openssl.so.27.0.2
/usr/lib/libpcre.so.1.2.8
/usr/lib/libstagefright_yuv.so
/usr/lib/libcheck.so.0
/usr/lib/libserviceutility.so
/usr/lib/libreadline.so
/usr/lib/libpcreposix.so.0.0.4
/usr/lib/libcom_err.so.2.1
/usr/lib/libstagefright_soft_aacdec.so
/usr/lib/libnbaio.so
/usr/lib/libz.so.1.2.11
/usr/lib/libz.so.1
/usr/lib/libpcrecpp.so
/usr/lib/libstagefright_foundation.so
/usr/lib/libVSC.so
/usr/lib/libsqlite3.so.0.8.6
/usr/lib/libnl-xfrm-3.so
/usr/lib/libgio-2.0.so.0.5000.2
/usr/lib/libgnutlsxx.so.28
/usr/lib/libpanel.so.5
/usr/lib/libsqlite3.so.0
/usr/lib/libform.so
/usr/lib/libbluetooth.so.3
/usr/lib/libexpat.so.1
/usr/lib/libdrmframework.so
/usr/lib/libturbojpeg.so
/usr/lib/libform.so.5
/usr/lib/libnl-genl-3.so.200.22.0
/usr/lib/libnetutils.so
/usr/lib/libsync.so
/usr/lib/libgalUtil.so
/usr/lib/libSDL2-2.0.so.0
/usr/lib/libbluetooth.so
/usr/lib/libdebugger.so
/usr/lib/libext2fs.so.2
/usr/lib/libstagefright_httplive.so
/usr/lib/libGLSLC.so
/usr/lib/libpanel.so
/usr/lib/libSDL2_image-2.0.so.0
/usr/lib/libcrypto.so.1.1
/usr/lib/libtinyalsa.so
/usr/lib/libjpeg.so
/usr/lib/libpng.so.3.37.0
/usr/lib/libmediautils.so
/usr/lib/libstagefright_omx.so
/usr/lib/libnl-nf-3.so.200.22.0
/usr/lib/libstagefright_soft_vorbisdec.so
/usr/lib/libstagefright_soft_rawdec.so
/usr/lib/libxtables.so.12.0.0
/usr/lib/libglib-2.0.so.0
/usr/lib/libffi.so.6.0.4
/usr/lib/libip6tc.so
/usr/lib/libgmodule-2.0.so
/usr/lib/libnettle.so
/usr/lib/libgnutlsxx.so
/usr/lib/libhistory.so.7.0
/usr/lib/libSDL2_image-2.0.so.0.2.3
/usr/lib/liblzo2.so.2
/usr/lib/libjhead.so
/usr/lib/libip6tc.so.0
/usr/lib/libexpat.so.1.6.2
/usr/lib/libasound.so.2
/usr/lib/libbacktrace.so
/usr/lib/libtasn1.so.6
/usr/lib/libstagefright.so
/usr/lib/libip6tc.so.0.1.0
/usr/lib/libmedialistener.so
/usr/lib/libstateMachineBinder.so
/usr/lib/libext2fs.so
/usr/lib/libhogweed.so.4
/usr/lib/libgthread-2.0.so.0.5000.2
/usr/lib/libaudiopolicyservice.so
/usr/lib/libbinder.so
/usr/lib/libip4tc.so.0
/usr/lib/libncurses.so
/usr/lib/libgnutls-openssl.so.27
/usr/lib/libblkid.so
/usr/lib/libip4tc.so
/usr/lib/libasound.so.2.0.0
/usr/lib/libuuid.so
/usr/lib/libutils.so
/usr/lib/libssl.so.1.0.0
/usr/lib/libjzipu.so
/usr/lib/libmp4v2.so
/usr/lib/libi2c.so.0
/usr/lib/libstagefright_hard_x264hwenc.so
/usr/lib/libdlog.so
/usr/lib/libgthread-2.0.so.0
/usr/lib/libswscale.so
/usr/lib/libwifi.so
/usr/lib/libSDL2_image.so
/usr/lib/libgio-2.0.so
/usr/lib/libcrypto.so
/usr/lib/libpcre.so
/usr/lib/hw
/usr/lib/hw/audio_policy.default.so
/usr/lib/hw/camera.default.so
/usr/lib/hw/audio.primary.default.so
/usr/lib/hw/local_time.default.so
/usr/lib/libfreetype.so
/usr/lib/libmenu.so
/usr/lib/libswscale.so.5
/usr/lib/libnl-3.so
/usr/lib/libFFTEm.so
/usr/lib/libaudioflinger.so
/usr/lib/libxtables.so
/usr/lib/libaudiohelper.so
/usr/lib/libgnutls.so.30
/usr/lib/libdbus-1.so
/usr/lib/libjpeg.so.8
/usr/lib/libglib-2.0.so
/usr/lib/libreadline.so.7
/usr/lib/libexif.so
/usr/lib/libgio-2.0.so.0
/usr/lib/libsonivox.so
/usr/lib/libgmodule-2.0.so.0.5000.2
/usr/lib/libe2p.so.2
/usr/lib/libssl.so
/usr/lib/libusbhost.so
/usr/lib/libaudioresampler.so
/usr/lib/libEGL.so
/usr/lib/libstdc++.so.6
/usr/lib/libss.so.2.0
/usr/lib/libhardware.so
/usr/lib/libSDL2-2.0.so.0.12.0
/usr/lib/libhistory.so
/usr/lib/libfreetype.so.6
/usr/lib/libffi.so.6
/usr/lib/libsqlite3.so
/usr/lib/libmount.so
/usr/lib/libhogweed.so
/usr/lib/libstagefright_hard_vlume.so
/usr/lib/libnl-nf-3.so.200
/usr/lib/libopenobex.so.1
/usr/lib/libhardware_legacy.so
/usr/lib/libopenobex.so
/usr/lib/libcheck.so.0.0.0
/usr/lib/libavutil.so
/usr/lib/libgmodule-2.0.so.0
/usr/lib/libcrypto.so.1.0.0
/usr/lib/cmake
/usr/lib/cmake/SDL2
/usr/lib/cmake/SDL2/sdl2-config-version.cmake
/usr/lib/cmake/SDL2/sdl2-config.cmake
/usr/lib/libgmp.so.10.3.2
/usr/lib/libSDL2.so
/usr/lib/libmp4v2.so.2
/usr/lib/libtasn1.so.6.5.5
/usr/lib/libnl-idiag-3.so.200.22.0
/usr/lib/libstagefrighthw.so
/usr/lib/libexpat.so
/usr/lib/libpanel.so.5.9
/usr/lib/libcamera_client.so
/usr/lib/libfreetype.so.6.3.20
/usr/lib/libmedialogservice.so
/usr/lib/libmenu.so.5
/usr/lib/libturbojpeg.so.0
/usr/lib/libcom_err.so
/usr/lib/libnl-3.so.200
/usr/lib/libmenu.so.5.9
/usr/lib/libjpeg.so.8.1.2
/usr/lib/libcutils.so
/usr/lib/libinfiray2_hard_x264hwenc.so
/usr/lib/libnl-idiag-3.so
/usr/lib/libwifiservice.so
/usr/lib/libstagefright_soft_amrnbenc.so
/usr/lib/libgnutls.so.30.14.11
/usr/lib/libjzhal.so
/usr/lib/libss.so
/usr/lib/libaudioroute.so
/usr/lib/libui.so
/usr/lib/libnl-nf-3.so
/usr/lib/libnl-genl-3.so
/usr/lib/libpcrecpp.so.0.0.1
/usr/lib/libmediaListenerBinder.so
/usr/lib/libunwind.so
/usr/lib/libmtpservice.so
/usr/lib/libjzhal.so.m200.3.4.0
/usr/lib/libturbojpeg.so.0.1.0
/usr/lib/libnl-xfrm-3.so.200
/usr/lib/libreadline.so.7.0
/usr/lib/libncurses.so.5
/usr/lib/libicuuc.so
/usr/lib/libpng.so.3
/usr/lib/libh264stream-test.so
/usr/lib/libi2c.so.0.1.0
/usr/lib/libcheck.so
/usr/lib/libncurses.so.5.9
/usr/lib/libOMX_Core.so
/usr/lib/libstagefright_rtsp.so
/usr/lib/libtinyalsa.so.1
/usr/lib/libcurses.so
/usr/lib/libatomic.so.1.1.0
/usr/lib/libiptc.so.0
/usr/lib/directfb-1.7-7
/usr/lib/directfb-1.7-7/gfxdrivers
/usr/lib/directfb-1.7-7/gfxdrivers/libdirectfb_gal.so
/usr/lib/libSDL2_ttf-2.0.so.0.14.1
/usr/lib/libaudioutils.so
/usr/lib/libstagefright_enc_common.so
/usr/lib/libCameraMark.so
/usr/lib/libVDK.so
/usr/lib/libnl-genl-3.so.200
/usr/lib/libform.so.5.9
/usr/lib/libip4tc.so.0.1.0
/usr/lib/libgnutlsxx.so.28.1.0
/usr/lib/libcamera_metadata.so
/usr/lib/libavutil.so.56
/usr/lib/libvorbisidec.so
/usr/lib/libmedialistenerservice.so
/usr/lib/libgnutls.so
/usr/lib/libspeexresampler.so
/usr/lib/libgmp.so.10
/usr/lib/libicui18n.so
/usr/lib/libstateMachineService.so
/usr/lib/libnl-3.so.200.22.0
/usr/lib/libSDL2_ttf.so
/usr/lib/libssl.so.1.1
/usr/lib/libasound.so
/usr/lib/libeffects.so
/usr/lib/libz.so
/usr/lib/liblzo2.so.2.0.0
/usr/lib/libgobject-2.0.so
/usr/lib/libdbus-1.so.3
/usr/lib/libGLESv2.so
/usr/lib/libresourcemanagerservice.so
/usr/lib/pkgconfig
/usr/lib/pkgconfig/sdl2.pc
/usr/lib/libbase.so
/usr/lib/libiptc.so.0.0.0
/usr/lib/libcom_err.so.2
/usr/lib/libnl-route-3.so.200
/usr/lib/libiptc.so
/usr/lib/libstagefright_avc_common.so
/usr/lib/libbluetooth.so.3.13.0
/usr/lib/libgnutls-openssl.so
/usr/lib/libmediaplayerservice.so
/usr/lib/libffi.so
/usr/lib/libntfs-3g.so.87.0.0
/usr/lib/libstagefright_soft_mp3dec.so
/usr/lib/libwifiBinder.so
/usr/lib/libmedia.so
/usr/lib/libstateMachine.so
/usr/lib/libcameraservice.so
/usr/lib/terminfo
/usr/lib/libext2fs.so.2.4
/usr/lib/libnl-idiag-3.so.200
/usr/lib/libaudiospdif.so
/usr/lib/libopus.so
/usr/lib/xtables
/usr/lib/xtables/libxt_physdev.so
/usr/lib/xtables/libip6t_REDIRECT.so
/usr/lib/xtables/libip6t_mh.so
/usr/lib/xtables/libip6t_SNAT.so
/usr/lib/xtables/libxt_state.so
/usr/lib/xtables/libebt_802_3.so
/usr/lib/xtables/libxt_tcpmss.so
/usr/lib/xtables/libxt_tos.so
/usr/lib/xtables/libip6t_REJECT.so
/usr/lib/xtables/libxt_set.so
/usr/lib/xtables/libxt_hashlimit.so
/usr/lib/xtables/libxt_dccp.so
/usr/lib/xtables/libxt_dscp.so
/usr/lib/xtables/libxt_AUDIT.so
/usr/lib/xtables/libip6t_ah.so
/usr/lib/xtables/libxt_udp.so
/usr/lib/xtables/libxt_ipcomp.so
/usr/lib/xtables/libxt_TCPOPTSTRIP.so
/usr/lib/xtables/libipt_CLUSTERIP.so
/usr/lib/xtables/libxt_statistic.so
/usr/lib/xtables/libxt_rpfilter.so
/usr/lib/xtables/libxt_SECMARK.so
/usr/lib/xtables/libxt_u32.so
/usr/lib/xtables/libipt_ECN.so
/usr/lib/xtables/libip6t_DNPT.so
/usr/lib/xtables/libxt_pkttype.so
/usr/lib/xtables/libebt_mark_m.so
/usr/lib/xtables/libipt_icmp.so
/usr/lib/xtables/libip6t_MASQUERADE.so
/usr/lib/xtables/libxt_TCPMSS.so
/usr/lib/xtables/libxt_NFLOG.so
/usr/lib/xtables/libxt_connmark.so
/usr/lib/xtables/libipt_realm.so
/usr/lib/xtables/libxt_NOTRACK.so
/usr/lib/xtables/libipt_REJECT.so
/usr/lib/xtables/libxt_cpu.so
/usr/lib/xtables/libxt_quota.so
/usr/lib/xtables/libxt_HMARK.so
/usr/lib/xtables/libip6t_ipv6header.so
/usr/lib/xtables/libxt_string.so
/usr/lib/xtables/libxt_CONNMARK.so
/usr/lib/xtables/libip6t_rt.so
/usr/lib/xtables/libxt_CHECKSUM.so
/usr/lib/xtables/libxt_DSCP.so
/usr/lib/xtables/libxt_devgroup.so
/usr/lib/xtables/libip6t_frag.so
/usr/lib/xtables/libip6t_LOG.so
/usr/lib/xtables/libxt_limit.so
/usr/lib/xtables/libxt_SET.so
/usr/lib/xtables/libxt_CT.so
/usr/lib/xtables/libxt_owner.so
/usr/lib/xtables/libxt_RATEEST.so
/usr/lib/xtables/libipt_SNAT.so
/usr/lib/xtables/libxt_CONNSECMARK.so
/usr/lib/xtables/libxt_NFQUEUE.so
/usr/lib/xtables/libxt_standard.so
/usr/lib/xtables/libxt_addrtype.so
/usr/lib/xtables/libipt_ULOG.so
/usr/lib/xtables/libxt_iprange.so
/usr/lib/xtables/libxt_policy.so
/usr/lib/xtables/libipt_TTL.so
/usr/lib/xtables/libipt_REDIRECT.so
/usr/lib/xtables/libxt_connbytes.so
/usr/lib/xtables/libxt_time.so
/usr/lib/xtables/libxt_MARK.so
/usr/lib/xtables/libip6t_hbh.so
/usr/lib/xtables/libxt_TOS.so
/usr/lib/xtables/libipt_ah.so
/usr/lib/xtables/libxt_conntrack.so
/usr/lib/xtables/libxt_TEE.so
/usr/lib/xtables/libip6t_HL.so
/usr/lib/xtables/libxt_LED.so
/usr/lib/xtables/libxt_tcp.so
/usr/lib/xtables/libxt_mark.so
/usr/lib/xtables/libxt_nfacct.so
/usr/lib/xtables/libxt_bpf.so
/usr/lib/xtables/libxt_mac.so
/usr/lib/xtables/libxt_ecn.so
/usr/lib/xtables/libxt_helper.so
/usr/lib/xtables/libxt_mangle.so
/usr/lib/xtables/libxt_osf.so
/usr/lib/xtables/libipt_DNAT.so
/usr/lib/xtables/libxt_IDLETIMER.so
/usr/lib/xtables/libip6t_dst.so
/usr/lib/xtables/libxt_cgroup.so
/usr/lib/xtables/libipt_NETMAP.so
/usr/lib/xtables/libipt_MASQUERADE.so
/usr/lib/xtables/libxt_ipvs.so
/usr/lib/xtables/libxt_CLASSIFY.so
/usr/lib/xtables/libxt_connlimit.so
/usr/lib/xtables/libipt_LOG.so
/usr/lib/xtables/libxt_recent.so
/usr/lib/xtables/libebt_log.so
/usr/lib/xtables/libxt_multiport.so
/usr/lib/xtables/libxt_rateest.so
/usr/lib/xtables/libxt_sctp.so
/usr/lib/xtables/libxt_TRACE.so
/usr/lib/xtables/libip6t_hl.so
/usr/lib/xtables/libxt_SYNPROXY.so
/usr/lib/xtables/libipt_ttl.so
/usr/lib/xtables/libxt_length.so
/usr/lib/xtables/libxt_socket.so
/usr/lib/xtables/libxt_cluster.so
/usr/lib/xtables/libip6t_SNPT.so
/usr/lib/xtables/libxt_TPROXY.so
/usr/lib/xtables/libxt_comment.so
/usr/lib/xtables/libip6t_eui64.so
/usr/lib/xtables/libip6t_NETMAP.so
/usr/lib/xtables/libip6t_DNAT.so
/usr/lib/xtables/libebt_ip.so
/usr/lib/xtables/libxt_esp.so
/usr/lib/xtables/libip6t_icmp6.so
/usr/lib/libnl.so
/usr/share
/usr/share/udhcpc
/usr/share/udhcpc/default.script
/usr/share/alsa
/usr/share/alsa/alsa.conf
/usr/share/alsa/topology
/usr/share/alsa/topology/sklrt286
/usr/share/alsa/topology/sklrt286/skl_i2s.conf
/usr/share/alsa/topology/bxtrt298
/usr/share/alsa/topology/bxtrt298/bxt_i2s.conf
/usr/share/alsa/topology/broadwell
/usr/share/alsa/topology/broadwell/broadwell.conf
/usr/share/alsa/sndo-mixer.alisp
/usr/share/alsa/ucm
/usr/share/alsa/ucm/VEYRON-I2S
/usr/share/alsa/ucm/VEYRON-I2S/HiFi.conf
/usr/share/alsa/ucm/VEYRON-I2S/VEYRON-I2S.conf
/usr/share/alsa/ucm/PandaBoardES
/usr/share/alsa/ucm/PandaBoardES/voiceCall
/usr/share/alsa/ucm/PandaBoardES/FMAnalog
/usr/share/alsa/ucm/PandaBoardES/hifi
/usr/share/alsa/ucm/PandaBoardES/voice
/usr/share/alsa/ucm/PandaBoardES/hifiLP
/usr/share/alsa/ucm/PandaBoardES/PandaBoardES.conf
/usr/share/alsa/ucm/PandaBoardES/record
/usr/share/alsa/ucm/chtrt5645
/usr/share/alsa/ucm/chtrt5645/HiFi.conf
/usr/share/alsa/ucm/chtrt5645/chtrt5645.conf
/usr/share/alsa/ucm/GoogleNyan
/usr/share/alsa/ucm/GoogleNyan/HiFi.conf
/usr/share/alsa/ucm/GoogleNyan/GoogleNyan.conf
/usr/share/alsa/ucm/skylake-rt286
/usr/share/alsa/ucm/skylake-rt286/Hdmi2
/usr/share/alsa/ucm/skylake-rt286/Hdmi1
/usr/share/alsa/ucm/skylake-rt286/HiFi
/usr/share/alsa/ucm/skylake-rt286/skylake-rt286.conf
/usr/share/alsa/ucm/tegraalc5632
/usr/share/alsa/ucm/tegraalc5632/tegraalc5632.conf
/usr/share/alsa/ucm/SDP4430
/usr/share/alsa/ucm/SDP4430/voiceCall
/usr/share/alsa/ucm/SDP4430/FMAnalog
/usr/share/alsa/ucm/SDP4430/SDP4430.conf
/usr/share/alsa/ucm/SDP4430/hifi
/usr/share/alsa/ucm/SDP4430/voice
/usr/share/alsa/ucm/SDP4430/hifiLP
/usr/share/alsa/ucm/SDP4430/record
/usr/share/alsa/ucm/DB410c
/usr/share/alsa/ucm/DB410c/HDMI
/usr/share/alsa/ucm/DB410c/DB410c.conf
/usr/share/alsa/ucm/DB410c/HiFi
/usr/share/alsa/ucm/PAZ00
/usr/share/alsa/ucm/PAZ00/HiFi.conf
/usr/share/alsa/ucm/PAZ00/Record.conf
/usr/share/alsa/ucm/PAZ00/PAZ00.conf
/usr/share/alsa/ucm/PandaBoard
/usr/share/alsa/ucm/PandaBoard/voiceCall
/usr/share/alsa/ucm/PandaBoard/FMAnalog
/usr/share/alsa/ucm/PandaBoard/PandaBoard.conf
/usr/share/alsa/ucm/PandaBoard/hifi
/usr/share/alsa/ucm/PandaBoard/voice
/usr/share/alsa/ucm/PandaBoard/hifiLP
/usr/share/alsa/ucm/PandaBoard/record
/usr/share/alsa/ucm/broadwell-rt286
/usr/share/alsa/ucm/broadwell-rt286/broadwell-rt286.conf
/usr/share/alsa/ucm/broadwell-rt286/HiFi
/usr/share/alsa/ucm/DAISY-I2S
/usr/share/alsa/ucm/DAISY-I2S/HiFi.conf
/usr/share/alsa/ucm/DAISY-I2S/DAISY-I2S.conf
/usr/share/alsa/alsa.conf.d
/usr/share/alsa/alsa.conf.d/README
/usr/share/alsa/pcm
/usr/share/alsa/pcm/front.conf
/usr/share/alsa/pcm/hdmi.conf
/usr/share/alsa/pcm/surround40.conf
/usr/share/alsa/pcm/iec958.conf
/usr/share/alsa/pcm/center_lfe.conf
/usr/share/alsa/pcm/dmix.conf
/usr/share/alsa/pcm/rear.conf
/usr/share/alsa/pcm/modem.conf
/usr/share/alsa/pcm/surround41.conf
/usr/share/alsa/pcm/surround51.conf
/usr/share/alsa/pcm/surround71.conf
/usr/share/alsa/pcm/side.conf
/usr/share/alsa/pcm/dsnoop.conf
/usr/share/alsa/pcm/surround50.conf
/usr/share/alsa/pcm/default.conf
/usr/share/alsa/pcm/surround21.conf
/usr/share/alsa/pcm/dpl.conf
/usr/share/alsa/cards
/usr/share/alsa/cards/CMI8338-SWIEC.conf
/usr/share/alsa/cards/PMacToonie.conf
/usr/share/alsa/cards/CMI8738-MC6.conf
/usr/share/alsa/cards/VIA686A.conf
/usr/share/alsa/cards/VXPocket.conf
/usr/share/alsa/cards/ENS1371.conf
/usr/share/alsa/cards/aliases.conf
/usr/share/alsa/cards/AACI.conf
/usr/share/alsa/cards/CMI8738-MC8.conf
/usr/share/alsa/cards/RME9652.conf
/usr/share/alsa/cards/CA0106.conf
/usr/share/alsa/cards/TRID4DWAVENX.conf
/usr/share/alsa/cards/SI7018.conf
/usr/share/alsa/cards/NFORCE.conf
/usr/share/alsa/cards/YMF744.conf
/usr/share/alsa/cards/Aureon51.conf
/usr/share/alsa/cards/AU8810.conf
/usr/share/alsa/cards/RME9636.conf
/usr/share/alsa/cards/SI7018
/usr/share/alsa/cards/SI7018/sndop-mixer.alisp
/usr/share/alsa/cards/SI7018/sndoc-mixer.alisp
/usr/share/alsa/cards/Audigy.conf
/usr/share/alsa/cards/USB-Audio.conf
/usr/share/alsa/cards/ICE1712.conf
/usr/share/alsa/cards/CS46xx.conf
/usr/share/alsa/cards/ATIIXP.conf
/usr/share/alsa/cards/ATIIXP-SPDMA.conf
/usr/share/alsa/cards/ENS1370.conf
/usr/share/alsa/cards/ICE1724.conf
/usr/share/alsa/cards/EMU10K1.conf
/usr/share/alsa/cards/Loopback.conf
/usr/share/alsa/cards/ICH-MODEM.conf
/usr/share/alsa/cards/VX222.conf
/usr/share/alsa/cards/FWSpeakers.conf
/usr/share/alsa/cards/VIA8233A.conf
/usr/share/alsa/cards/GUS.conf
/usr/share/alsa/cards/CMI8788.conf
/usr/share/alsa/cards/Aureon71.conf
/usr/share/alsa/cards/AU8820.conf
/usr/share/alsa/cards/FM801.conf
/usr/share/alsa/cards/VIA8233.conf
/usr/share/alsa/cards/PMac.conf
/usr/share/alsa/cards/Echo_Echo3G.conf
/usr/share/alsa/cards/FireWave.conf
/usr/share/alsa/cards/AU8830.conf
/usr/share/alsa/cards/Audigy2.conf
/usr/share/alsa/cards/ICH.conf
/usr/share/alsa/cards/SB-XFi.conf
/usr/share/alsa/cards/PS3.conf
/usr/share/alsa/cards/ES1968.conf
/usr/share/alsa/cards/VXPocket440.conf
/usr/share/alsa/cards/ATIIXP-MODEM.conf
/usr/share/alsa/cards/Maestro3.conf
/usr/share/alsa/cards/ICH4.conf
/usr/share/alsa/cards/aliases.alisp
/usr/share/alsa/cards/EMU10K1X.conf
/usr/share/alsa/cards/CMI8338.conf
/usr/share/alsa/cards/VIA8237.conf
/usr/share/alsa/cards/PC-Speaker.conf
/usr/share/alsa/cards/HDA-Intel.conf
/usr/share/gettext
/usr/share/gettext/its
/usr/share/gettext/its/gschema.its
/usr/share/gettext/its/gschema.loc
/usr/share/bash-completion
/usr/share/bash-completion/completions
/usr/share/bash-completion/completions/gapplication
/usr/share/bash-completion/completions/gresource
/usr/share/bash-completion/completions/gsettings
/usr/share/bash-completion/completions/uuidgen
/usr/share/bash-completion/completions/gdbus
/usr/share/aclocal
/usr/share/aclocal/gsettings.m4
/usr/share/aclocal/glib-gettext.m4
/usr/share/aclocal/glib-2.0.m4
/usr/share/gdb
/usr/share/gdb/auto-load
/usr/share/gdb/auto-load/libglib-2.0.so.0.4000.0-gdb.py
/usr/share/glib-2.0
/usr/share/glib-2.0/gdb
/usr/share/glib-2.0/gdb/glib_gdb.py
/usr/share/glib-2.0/gdb/gobject_gdb.py
/usr/share/glib-2.0/gdb/glib.py
/usr/share/images
/usr/share/images/PS-D效果图
/usr/share/images/PS-D效果图/1开关机
/usr/share/images/PS-D效果图/1开关机/图片36.png
/usr/share/images/PS-D效果图/1开关机/图片1.png
/usr/share/images/PS-D效果图/0全部
/usr/share/images/PS-D效果图/0全部/图片30.png
/usr/share/images/PS-D效果图/0全部/图片11.png
/usr/share/images/PS-D效果图/0全部/图片12.png
/usr/share/images/PS-D效果图/0全部/图片16.png
/usr/share/images/PS-D效果图/0全部/图片5.png
/usr/share/images/PS-D效果图/0全部/图片4.png
/usr/share/images/PS-D效果图/0全部/图片15.png
/usr/share/images/PS-D效果图/0全部/图片27.png
/usr/share/images/PS-D效果图/0全部/图片19.png
/usr/share/images/PS-D效果图/0全部/图片10.png
/usr/share/images/PS-D效果图/0全部/图片17.png
/usr/share/images/PS-D效果图/0全部/图片13.png
/usr/share/images/PS-D效果图/0全部/图片7.png
/usr/share/images/PS-D效果图/0全部/图片6.png
/usr/share/images/PS-D效果图/0全部/图片33.png
/usr/share/images/PS-D效果图/0全部/图片23.png
/usr/share/images/PS-D效果图/0全部/图片34.png
/usr/share/images/PS-D效果图/0全部/图片29.png
/usr/share/images/PS-D效果图/0全部/图片36.png
/usr/share/images/PS-D效果图/0全部/图片8.png
/usr/share/images/PS-D效果图/0全部/图片22.png
/usr/share/images/PS-D效果图/0全部/图片26.png
/usr/share/images/PS-D效果图/0全部/图片1.png
/usr/share/images/PS-D效果图/0全部/图片9.png
/usr/share/images/PS-D效果图/0全部/图片24.png
/usr/share/images/PS-D效果图/0全部/图片21.png
/usr/share/images/PS-D效果图/0全部/图片2.png
/usr/share/images/PS-D效果图/0全部/图片18.png
/usr/share/images/PS-D效果图/0全部/图片31.png
/usr/share/images/PS-D效果图/0全部/图片20.png
/usr/share/images/PS-D效果图/0全部/图片32.png
/usr/share/images/PS-D效果图/0全部/图片25.png
/usr/share/images/PS-D效果图/0全部/图片3.png
/usr/share/images/PS-D效果图/0全部/图片14.png
/usr/share/images/PS-D效果图/0全部/图片28.png
/usr/share/images/PS-D效果图/0全部/图片35.png
/usr/share/images/PS-D效果图/2主界面
/usr/share/images/PS-D效果图/2主界面/图片4.png
/usr/share/images/PS-D效果图/2主界面/图片2.png
/usr/share/images/PS-D效果图/2主界面/图片3.png
/usr/share/images/PS-D效果图/菜单
/usr/share/images/PS-D效果图/菜单/选择6文件管理
/usr/share/images/PS-D效果图/菜单/选择6文件管理/图片21.png
/usr/share/images/PS-D效果图/菜单/选择6文件管理/图片20.png
/usr/share/images/PS-D效果图/菜单/图片5.png
/usr/share/images/PS-D效果图/菜单/选择7快速设置
/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片23.png
/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片22.png
/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片26.png
/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片24.png
/usr/share/images/PS-D效果图/菜单/选择7快速设置/图片25.png
/usr/share/images/PS-D效果图/菜单/图片19.png
/usr/share/images/PS-D效果图/菜单/选择5目标距离
/usr/share/images/PS-D效果图/菜单/选择5目标距离/图片18.png
/usr/share/images/PS-D效果图/菜单/选择3电子变倍
/usr/share/images/PS-D效果图/菜单/选择3电子变倍/图片12.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片30.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片27.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片33.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片34.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片29.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片31.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片32.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片28.png
/usr/share/images/PS-D效果图/菜单/选择8高级设置/图片35.png
/usr/share/images/PS-D效果图/菜单/选择2红外图像
/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片11.png
/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片10.png
/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片7.png
/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片8.png
/usr/share/images/PS-D效果图/菜单/选择2红外图像/图片9.png
/usr/share/images/PS-D效果图/菜单/选择1屏幕亮度
/usr/share/images/PS-D效果图/菜单/选择1屏幕亮度/图片6.png
/usr/share/images/PS-D效果图/菜单/选择4分划设置
/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片16.png
/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片15.png
/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片17.png
/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片13.png
/usr/share/images/PS-D效果图/菜单/选择4分划设置/图片14.png
/usr/share/images/图片11.png
/usr/share/images/背景.png
/usr/share/images/beijing3.png
/usr/share/images/beijing.png
/usr/share/images/PS-D960_768切图
/usr/share/images/PS-D960_768切图/常显
/usr/share/images/PS-D960_768切图/常显/4电池-4.png
/usr/share/images/PS-D960_768切图/常显/4电池-1.png
/usr/share/images/PS-D960_768切图/常显/0顶栏背景.png
/usr/share/images/PS-D960_768切图/常显/1可见光图像.png
/usr/share/images/PS-D960_768切图/常显/其它
/usr/share/images/PS-D960_768切图/常显/其它/z西.png
/usr/share/images/PS-D960_768切图/常显/其它/z北.png
/usr/share/images/PS-D960_768切图/常显/其它/z南.png
/usr/share/images/PS-D960_768切图/常显/其它/z东.png
/usr/share/images/PS-D960_768切图/常显/0顶栏背景-遮罩.png
/usr/share/images/PS-D960_768切图/常显/0右栏背景-遮罩.png
/usr/share/images/PS-D960_768切图/常显/1白热.png
/usr/share/images/PS-D960_768切图/常显/1红热.png
/usr/share/images/PS-D960_768切图/常显/1伪彩.png
/usr/share/images/PS-D960_768切图/常显/2电子变倍背景.png
/usr/share/images/PS-D960_768切图/常显/3录像.png
/usr/share/images/PS-D960_768切图/常显/0右栏背景.png
/usr/share/images/PS-D960_768切图/常显/数字
/usr/share/images/PS-D960_768切图/常显/数字/8.png
/usr/share/images/PS-D960_768切图/常显/数字/负号.png
/usr/share/images/PS-D960_768切图/常显/数字/1.png
/usr/share/images/PS-D960_768切图/常显/数字/6.png
/usr/share/images/PS-D960_768切图/常显/数字/点.png
/usr/share/images/PS-D960_768切图/常显/数字/2.png
/usr/share/images/PS-D960_768切图/常显/数字/度.png
/usr/share/images/PS-D960_768切图/常显/数字/9.png
/usr/share/images/PS-D960_768切图/常显/数字/4.png
/usr/share/images/PS-D960_768切图/常显/数字/7.png
/usr/share/images/PS-D960_768切图/常显/数字/5.png
/usr/share/images/PS-D960_768切图/常显/数字/0.png
/usr/share/images/PS-D960_768切图/常显/数字/3.png
/usr/share/images/PS-D960_768切图/常显/4电池.png
/usr/share/images/PS-D960_768切图/常显/5俯仰角三角.png
/usr/share/images/PS-D960_768切图/常显/0目标距离.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-2.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-1.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-4.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/7.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-8.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/5.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-0.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/8-6.png
/usr/share/images/PS-D960_768切图/常显/电子变倍数字/3.png
/usr/share/images/PS-D960_768切图/常显/4电池-2.png
/usr/share/images/PS-D960_768切图/常显/4电池-5.png
/usr/share/images/PS-D960_768切图/常显/3拍照.png
/usr/share/images/PS-D960_768切图/常显/1红外图像.png
/usr/share/images/PS-D960_768切图/常显/0底栏背景 右.png
/usr/share/images/PS-D960_768切图/常显/分划
/usr/share/images/PS-D960_768切图/常显/分划/8.png
/usr/share/images/PS-D960_768切图/常显/分划/1.png
/usr/share/images/PS-D960_768切图/常显/分划/6.png
/usr/share/images/PS-D960_768切图/常显/分划/2.png
/usr/share/images/PS-D960_768切图/常显/分划/4.png
/usr/share/images/PS-D960_768切图/常显/分划/7.png
/usr/share/images/PS-D960_768切图/常显/分划/5.png
/usr/share/images/PS-D960_768切图/常显/分划/3.png
/usr/share/images/PS-D960_768切图/常显/数码管数字
/usr/share/images/PS-D960_768切图/常显/数码管数字/8.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/1.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/6.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/2.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/9.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/4.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/7.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/5.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/0.png
/usr/share/images/PS-D960_768切图/常显/数码管数字/3.png
/usr/share/images/PS-D960_768切图/常显/1黑热.png
/usr/share/images/PS-D960_768切图/常显/5方位角背景.png
/usr/share/images/PS-D960_768切图/常显/5方位角三角.png
/usr/share/images/PS-D960_768切图/常显/5俯仰背景.png
/usr/share/images/PS-D960_768切图/常显/2电子变倍背景 x .png
/usr/share/images/PS-D960_768切图/常显/4电池-3.png
/usr/share/images/PS-D960_768切图/常显/0底栏背景 左.png
/usr/share/images/PS-D960_768切图/中.png
/usr/share/images/PS-D960_768切图/俄.png
/usr/share/images/PS-D960_768切图/开关机
/usr/share/images/PS-D960_768切图/开关机/1激光.png
/usr/share/images/PS-D960_768切图/开关机/其它
/usr/share/images/PS-D960_768切图/开关机/其它/1.bmp
/usr/share/images/PS-D960_768切图/开关机/3进度条-2.png
/usr/share/images/PS-D960_768切图/开关机/1电子罗盘.png
/usr/share/images/PS-D960_768切图/开关机/3进度条.png
/usr/share/images/PS-D960_768切图/开关机/4关.png
/usr/share/images/PS-D960_768切图/开关机/3进度条-1.png
/usr/share/images/PS-D960_768切图/开关机/2异常.png
/usr/share/images/PS-D960_768切图/开关机/1省略号.png
/usr/share/images/PS-D960_768切图/开关机/1模组.png
/usr/share/images/PS-D960_768切图/开关机/1电池.png
/usr/share/images/PS-D960_768切图/开关机/4关-1.png
/usr/share/images/PS-D960_768切图/开关机/1背景.png
/usr/share/images/PS-D960_768切图/开关机/2正常.png
/usr/share/images/PS-D960_768切图/语言切换.png
/usr/share/images/PS-D960_768切图/法.png
/usr/share/images/PS-D960_768切图/英语.png
/usr/share/images/PS-D960_768切图/菜单
/usr/share/images/PS-D960_768切图/菜单/7快速设置.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/3罗盘.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/1.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/2.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/ON.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠/off.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化
/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化/选中.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/7格式化/背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/3罗盘
/usr/share/images/PS-D960_768切图/菜单/8高级设置/3罗盘/1.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/8.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/1.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/6.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/2.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/9.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/4.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/7.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/5.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/0.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/3.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/数字/-.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/确定.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/四字框.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/取消.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/1时间/两字框.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/2休眠.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/8故障
/usr/share/images/PS-D960_768切图/菜单/8高级设置/8故障/背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/8故障.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复
/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复/选中.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/6恢复/背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/0背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/退出.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/菜单.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/Y.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/X.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/cancel.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/确定.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/分划.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/4盲元/add.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/射表.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/选中.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/8.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/1.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/6.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/2.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/9.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/4.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/7.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/5.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/0.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/数字/3.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/选择位数.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/射表背景.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/枪型.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置/5射表/弹种.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划位置.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1取消.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/2正号.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1确定.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0数值背景.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0X标尺.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/2分划.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0Y标尺.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1X.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/2负号.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1数值选中.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/1Y.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划调整/0菜单.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/背景.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划样式.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划亮度.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/绿色.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/蓝色.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/白色.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/黑色.png
/usr/share/images/PS-D960_768切图/菜单/4分化设置/分划颜色/红色.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/8.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/1.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/6.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/2.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/9.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/4.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/7.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/5.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/0.png
/usr/share/images/PS-D960_768切图/菜单/1屏幕亮度/3.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像
/usr/share/images/PS-D960_768切图/菜单/2红外图像/背景校正-1.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/4背景校正.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/1对比度.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/3图像模式.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/2图像增强.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/0背景.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/0亮度.png
/usr/share/images/PS-D960_768切图/菜单/2红外图像/未标题-1.png
/usr/share/images/PS-D960_768切图/菜单/3电子变倍
/usr/share/images/PS-D960_768切图/菜单/3电子变倍/变倍-1.png
/usr/share/images/PS-D960_768切图/菜单/8高级设置.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理
/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-0.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-2.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/2图片.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/2视频.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/es.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/背景.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/fra.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/rus.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/en.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/帮助/cn.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/8.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/6.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/2.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/9.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/4.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/7.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/5.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/0.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/数字/3.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/0文件选框.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3下一张.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3下一张-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快进-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/4文件名背景.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3上一张-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0暂停-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/1播放进度条-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/1播放进度条.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/3上一张.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0播放-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快进.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0播放.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快退-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/0暂停.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/播放/2快退.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-4.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/0大背景.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-3.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/1-1.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/0菜单.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/0视频播放图标.png
/usr/share/images/PS-D960_768切图/菜单/6文件管理/2帮助.png
/usr/share/images/PS-D960_768切图/菜单/3变倍设置.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离
/usr/share/images/PS-D960_768切图/菜单/5目标距离/选中.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/背景.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/8.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/1.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/6.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/2.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/9.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/4.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/7.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/5.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/0.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/数字/3.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/确认.png
/usr/share/images/PS-D960_768切图/菜单/5目标距离/取消.png
/usr/share/images/PS-D960_768切图/菜单/0背景.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整
/usr/share/images/PS-D960_768切图/菜单/7快速调整/OFF.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/快速瞄准.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/蓝牙.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/wifi.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/on.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/视频传输.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/OFF-1.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/分划.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/0背景.png
/usr/share/images/PS-D960_768切图/菜单/7快速调整/on-1.png
/usr/share/images/PS-D960_768切图/西.png
/usr/share/dbus-1
/usr/share/dbus-1/system-services
/usr/share/dbus-1/system-services/org.bluez.service
/usr/share/dbus-1/session.conf
/usr/share/dbus-1/system.conf
/usr/share/terminfo
/usr/share/terminfo/p
/usr/share/terminfo/p/putty
/usr/share/terminfo/p/putty-vt100
/usr/share/terminfo/a
/usr/share/terminfo/a/ansi
/usr/share/terminfo/v
/usr/share/terminfo/v/vt102
/usr/share/terminfo/v/vt200
/usr/share/terminfo/v/vt100-putty
/usr/share/terminfo/v/vt100
/usr/share/terminfo/v/vt220
/usr/share/terminfo/s
/usr/share/terminfo/s/screen
/usr/share/terminfo/l
/usr/share/terminfo/l/linux
/usr/share/terminfo/x
/usr/share/terminfo/x/xterm-xfree86
/usr/share/terminfo/x/xterm
/usr/share/terminfo/x/xterm-color
/usr/lib32
/usr/data
/usr/data/ota_res
/usr/data/ota_res/update_play.sh
/usr/data/ota_res/recovery.conf
/usr/data/ota_res/update_stage.sh
/usr/data/ota_res/key
/usr/data/ota_res/key/key.pub
/usr/data/ota_res/update.sh
/usr/data/ota_res/recovery
/usr/data/libcutils.so
/usr/data/wpa_supplicant
/usr/data/UPDATE_RET
/usr/data/lost+found
/usr/data/STAGE
/usr/data/firmware
/usr/data/firmware/wifimac.txt
/usr/data/firmware/fw_43438_a1.bin
/usr/data/firmware/nv_43438_a1.cal
/usr/data/wpa_supplicant.conf
/lib32
/data
/data/database
/data/database/mtp.db
/var
/var/run
/var/tmp
/var/www
/var/www/update.html
/var/www/halmic.html
/var/www/flashing.html
/var/www/fail.html
/var/www/art.html
/var/www/update.asp
/var/www/style.css
/var/www/halfpga.html
/var/www/404.html
/var/www/cgi-bin
/var/www/cgi-bin/setdevicecfg.cgi
/var/www/cgi-bin/setpasswd
/var/www/cgi-bin/pushfile.cgi
/var/www/cgi-bin/cgictest.cgi
/var/www/cgi-bin/trio_tmr
/var/www/cgi-bin/productInfo2
/var/www/cgi-bin/setssid
/var/www/cgi-bin/version
/var/www/cgi-bin/getpasswd
/var/www/cgi-bin/pushfpga.cgi
/var/www/cgi-bin/getssid
/var/www/cgi-bin/productVersion
/var/www/cgi-bin/cgijson.cgi
/var/www/mac.html
/var/lock
/var/lib
/var/lib/misc
/var/lib/dbus
/var/spool
/var/cache
/var/log
/storage
/storage/20251116
/storage/20251116/IMG_20251116120335_00.jpeg
/storage/20251116/IMG_20251116120341_00.jpeg
/storage/20251116/IMG_20251116120338_00.jpeg
/storage/20251116/IMG_20251116120340_00.jpeg
/storage/20251111
/storage/20251111/IMG_20251111213413_00.jpeg
/storage/20251111/IMG_20251111215835_00.jpeg
/storage/20251111/IMG_20251111175230_00.jpeg
/storage/20251111/IMG_20251111195309_00.jpeg
/storage/20251111/IMG_20251111220008_00.jpeg
/storage/update
/storage/20251112
/storage/20251112/IMG_20251112174501_00.jpeg
/storage/20251112/IMG_20251112182059_00.jpeg
/storage/20251112/IMG_20251112182058_00.jpeg
/storage/20251112/VID_20251112174450.mp4
/storage/20251112/IMG_20251112174435_00.jpeg
/storage/20251112/IMG_20251112182057_00.jpeg
```
