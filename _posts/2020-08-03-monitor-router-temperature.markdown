---
layout: post
title:  "Monitoring temperatures on your wireless router with Prometheus"
categories: monitoring router prometheus
date:   2020-08-03 20:35:42 +0530
series: "router-monitoring"
---

This is third in [series of posts]({{ site.url }}/series/router-monitoring) on monitoring your wireless router. In this post we will setup a script to pull
temperature information from the router and expose it to prometheus using node exporter.  Check [this post]({% post_url 2020-07-30-router-series-1 %})
if you need instructions on how to setup node exporter on your router.

> **Caution:** There is very little chance that these instructions will be portable as-is to any firmware other than Asus WRT Merlin.

I have an [Asus RT-AC68U](https://www.asus.com/in/Networking/RTAC68U/) with [AsusWRT-Merlin](https://www.asuswrt-merlin.net/) firmware installed on it. 
This allows me to setup cron jobs and custom scripts on the router which is required to setup the monitoring. The instructions here should work on other 
Asus routers with Meriln firmware but are unlikely to work on other firmwares.

For pulling and persisting the metrics I am be using [Prometheus](https://prometheus.io) which is an open-source monitoring solution. I have it installed
on a home server which is up 24x7 and connected to the router. A static IP is configured for the server which makes things convenient but the setup
will also work with a dynamic IP address assignment.

## Prerequisites

You need to be able to SSH into the router and have node_exporter installed on it.  Check [here]({% post_url 2020-07-30-router-series-1 %})
for instructions on how to setup node exporter on your router. You need to enable Custom JFFS scripts from `Administration > System` in your WebUI.

For the sake of this post I am assuming that we have an external USB stick connected to the router which is mounted at `/mnt/<disk>` and we have following 
directory structure created:

<pre>
/mnt/&lt;disk&gt;
|- bin
|- etc
  |- services
|- var
  |- log
  |- run
  |- lib
    |- node_exporter
|- home
  |- user
|- tmp
</pre>

You can create the directory structure using following command. **(Change \<disk\> to the name of your disk.)**

{% highlight bash %}
mkdir -p /mnt/<disk>/{bin,etc/services,home/user,var/{log,run,lib/node_exporter},tmp}
{% endhighlight %}

## Setting up the script

We will be storing the data extraction script on the external drive. Create a file `/mnt/<disk>/bin/hw-temp` with following content in it.

{% highlight bash %}
#!/bin/sh

# Note the start time of the script.
START="$(date +%s)"

cpu_temp=`cat /proc/dmu/temperature | grep -Eo '[0-9]+'`
hz_2_4_temp_raw=` wl -i eth1 phy_tempsense | grep -Eo '^[0-9]+'`
hz_2_4_temp=`expr 20 + $hz_2_4_temp_raw / 2`
hz_5_temp_raw=` wl -i eth2 phy_tempsense | grep -Eo '^[0-9]+'`
hz_5_temp=`expr 20 + $hz_5_temp_raw / 2`

# Adjust as needed.
TEXTFILE_COLLECTOR_DIR="/tmp/mnt/<disk>/var/lib/node_exporter"

# Write out metrics to a temporary file.
END="$(date +%s)"
cat << EOF > "$TEXTFILE_COLLECTOR_DIR/temperature.prom.$$"
router_temperature_celcius{type="cpu"} $cpu_temp
router_temperature_celcius{type="antenna_5"} $hz_5_temp
router_temperature_celcius{type="antenna_24"} $hz_2_4_temp
router_temperature_duration_seconds $(($END - $START))
router_temperature_last_run_seconds $END
EOF

# Rename the temporary file atomically.
# This avoids the node exporter seeing half a file.
mv "$TEXTFILE_COLLECTOR_DIR/temperature.prom.$$" \
  "$TEXTFILE_COLLECTOR_DIR/temperature.prom"

{% endhighlight %}

The script will report following 4 metrics:

1. `router_temperature_duration_seconds`: runtime of `nvram_free` script in seconds
2. `router_temperature_last_run_seconds`: unix timestamp of last run
3. `router_temperature_celcius`: current temperature. This metric is exported with a label `type`:
    1. `cpu`: Current CPU temperature
    2. `antenna_5`: 5G antenna temperature
    3. `antenna_24`: 2.4G antenna temperature

Mark this script as executable.

{% highlight bash %}
chmod 755 /mnt/<disk>/bin/hw-temp
{% endhighlight %}

Run the script once and check if it creates the text file with metrics in `/mnt/<disk>/var/lib/node_exporter/temperature.prom`. Following is a sample content from file.

<pre>
router_temperature_celcius{type="cpu"} 73
router_temperature_celcius{type="antenna_5"} 54
router_temperature_celcius{type="antenna_24"} 51
router_temperature_duration_seconds 0
router_temperature_last_run_seconds 1596466501
</pre>

Next we setup cron job to run this script every minute using the `cru` command.

{% highlight bash %}
# Usage: cru a <unique_id> '<cron_expression> <command>'
cru a tmp '* * * * * /mnt/<disk>/bin/hw-temp'
{% endhighlight %}

The cron job entry will be gone after reboot. To persist it we will need to add the command to user script which is called on boot. Since we
are storing the `hw-temp` command on external disk we also need to ensure that the disk is mounted before we add the cron job. We will be 
using the `post-mount` script. Create a file `/jffs/scripts/post-mount` if it not already exists. Add following content to the file. The 
`post-mount` script is passed the mountpoint as the first argument.

{% highlight bash %}
#!/bin/sh

MOUNTPOINT=$1

if [[ $MOUNTPOINT == "/tmp/mnt/<disk>" ]]; then
    echo "$1 Mounted" >> "$1/var/log/syslog"
    cru a nvr '* * * * * /mnt/<disk>/bin/hw-temp'
fi
{% endhighlight %}

Now we have a script which periodically checks the temperature and reports the data in prometheus format in a file. The only thing remaining is to
tell our node exporter to pickup the file on every scrape. To do this change the startup command for node exporter by adding `--collector.textfile.directory`
parameter. If you followed the [first post]({% post_url 2020-07-30-router-series-1 %}) to setup the node exporter following is the updated service script.

{% highlight bash %}
#!/bin/sh

ROOT_DIR=$1
CMD="$1/bin/node_exporter"
LOG="$1/var/log/node_exporter.log"

pidof node_exporter

if [[ $? -ne 0 ]] ; then
    $CMD --web.listen-address=":9101" --collector.textfile.directory=/tmp/mnt/<disk>/var/lib/node_exporter >> "$LOG" 2>&1 &
else
    echo "Already running"
fi
{% endhighlight %}

To restart the node exporter you can use following commands.

{% highlight bash %}
pidof node_exporter 
# Note the PID printed here

kill -15 <pid_of_node_exporter>
/mnt/<disk>/etc/services/node_exporter.service
{% endhighlight %}

Thats it! You can visit `http://<your_router_ip>:9101/metrics` and check if `router_temperature_` metrics are being reported.
