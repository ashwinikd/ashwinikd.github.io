---
layout: post
title:  "Connecting to WiFi on Ubuntu 20.04 Headless Server"
date:   2020-07-28 16:14:42 +0530
categories: ubuntu server ops networking ubuntu-falcon
---
If you are running headless ubuntu server you will not be connected to the wireless automatically.
The instructions below have been tested on Ubuntu 20.04 (Falcon) Server but should work with Ubuntu 18.04
(Bionic) and 16.04 (Xenial).

We will be following these steps:

1. Installing tools and utilities
2. Finding the wireless interface name
3. Scanning for the wireless connections
4. Setting up WPA config file
5. Connecting to the wireless network
6. Obtain IP address
7. Setup auto connect on boot

## Requirements

You will need following utilities to connect to WiFi:

- `wireless-tools`: Set of utilities for interacting with wireless devices 
- `wpa_supplicant`: Required to connect using WPA/WPA2 authentication

Install the requirement using following commands:

{% highlight bash %}
sudo apt install wireless-tools wpa_supplicant
{% endhighlight %}

## Setup

Run `iwconfig` to get the interface name for your wirless device.

{% highlight bash %}
iwconfig
{% endhighlight %}

This will give you output similar to following:

<pre>
enp4s0    no wireless extensions.

<b>wlp5s0</b>    IEEE 802.11  ESSID:off/any  
          Mode:Managed  Access Point: Not-Associated   Tx-Power=0 dBm
          Retry short limit:7   RTS thr:off   Fragment thr:off
          Power Management:on

lo        no wireless extensions.

enp0s31f6  no wireless extensions.
</pre>

Output shows us that the interface `wlp5s0` supports wireless connectivity. Next step is to scan the network.

{% highlight bash %}
sudo iw <interface> scan
{% endhighlight %}

The output will be similar to following:

<pre>
---- truncated ----
BSS ab:cd:ef:01:23:45(on wlp5s0) -- associated
        last seen: 12274.418s [boottime]
        TSF: 573122111163 usec (6d, 15:12:02)
        freq: 5200
        beacon interval: 100 TUs
        capability: ESS Privacy RadioMeasure (0x1011)
        signal: -36.00 dBm
        last seen: 1036 ms ago
        Information elements from Probe Response frame:
        <b>SSID: Uchiha_5G</b>
        Supported rates: 6.0* 9.0 12.0* 18.0 24.0* 36.0 48.0 54.0 
        <b>RSN</b>:     * Version: 1
                 * Group cipher: CCMP
                 * Pairwise ciphers: CCMP
                 * Authentication suites: PSK
                 * Capabilities: 16-PTKSA-RC 1-GTKSA-RC (0x000c)
        BSS Load:
                 * station count: 7
                 * channel utilisation: 52/255
                 * available admission capacity: 0 [*32us]
---- truncated ----
</pre>

The wireless network name will be mentioned under `SSID`. The `RSN` section signifies that the authentication
is using WPA/WPA2 and Authentication suites is `PSK` which means Pre-Shared Key (a password). Now we create
a configuration file for password using following command. The command will wait for your password on the
shell (**Replace `WIFI_PASSWORD` with your own password**):

{% highlight bash %}
wpa_passphrase Uchiha_5G | sudo tee /etc/wpa_supplicant.conf
WIFI_PASSWORD
{% endhighlight %}

Run following command to connect to the Wireless network.

{% highlight bash %}
wpa_supplicant -c /etc/wpa_supplicant.conf -i <interface>
{% endhighlight %}

This will attempt to connect to the network and report if it faces any problems. This might take around a minute.
If you see `CTRL-EVENT-CONNECTED - Connection to ab:cd:ef:01:23:45 completed` then all is well. You can do 
`CTRL-C` to exit to terminal. To run the command in background supply `-B` flag:

{% highlight bash %}
wpa_supplicant -B -c /etc/wpa_supplicant.conf -i <interface>
{% endhighlight %}

Lets verify if we are actually connected:

{% highlight bash %}
iw <interface> link
{% endhighlight %}

It should show an output similar to following:

```text
Connected to ab:cd:ef:01:23:45 (on wlp5s0)
  SSID: Uchiha_5G
  freq: 5200
  RX: 1427916 bytes (13735 packets)
  TX: 235064 bytes (1390 packets)
  signal: -41 dBm
  rx bitrate: 6.0 MBit/s
  tx bitrate: 866.7 MBit/s VHT-MCS 9 80MHz short GI VHT-NSS 2

  bss flags:    short-slot-time
  dtim period:  3
  beacon int:   100
```

If your output shows `Not connected` then you should check if `wpa_supplicant` command is running in
background using `ps aux | grep wpa`. You can run the `wpa_supplicant` command without the `-B` flag
to check what is going wrong.

We are now connected to WiFi but we also need to get an IP Address from router. To do this we use
the `dhclient` utility.

{% highlight bash %}
sudo /sbin/dhclient <interface>
{% endhighlight %}

Once this is done we can check the status using `ifconfig` to see if we are connected and have an
ip address assigned. You can also run a ping-test to check if you have internet.

## Auto-connect on boot

To setup auto-connect on boot we use systemd to run the same commands in correct order. Create a file
`/etc/systemd/system/connect-wireless.service` with following content in it:

{% highlight systemd %}
[Unit]
Description=WPA supplicant for connecting to My WiFi
After=wpa_supplicant.service
Wants=network.target
IgnoreOnIsolate=true

[Service]
ExecStart=/sbin/wpa_supplicant -c /etc/wpa_supplicant.conf -i <interface>

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Create a service file `/etc/systemd/system/dhclient.service` for getting IP address after `wpa_supplicant`.
Notice the `After` directive in the `[Unit]` section. If you are using different service name you should
update this directive with the correct name to your `wpa_supplicant` service.

{% highlight systemd %}
[Unit]
Description= DHCP Client
Before=network.target
After=connect-wireless.service

[Service]
Type=forking
ExecStart=/sbin/dhclient wlp5s0 -v
ExecStop=/sbin/dhclient wlp5s0 -r

[Install]
WantedBy=multi-user.target
{% endhighlight %}

Now we enable the services:

{% highlight bash %}
sudo systemctl enable connect-wireless.service
sudo systemctl enable dhclient.service
{% endhighlight %}

You can reboot and test if your services are working and connecting to wireless correctly.
