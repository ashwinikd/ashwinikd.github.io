---
layout: post
title:  "Monitoring your wireless router with Prometheus Node Exporter"
date:   2020-07-30 16:14:42 +0530
categories: monitoring router prometheus
---

This is first in series of posts on monitoring your wireless router. In this post we will setup [node_exporter](https://github.com/prometheus/node_exporter) 
on the router which will allow us to pull basic performance metrics. I have an [Asus RT-AC68U](https://www.asus.com/in/Networking/RTAC68U/) with
[AsusWRT-Merlin](https://www.asuswrt-merlin.net/) firmware installed on it. This allows me to setup cron jobs and custom scripts on the router which 
is required to setup the monitoring. You can check out the complete list of [additional features](https://www.asuswrt-merlin.net/features) that Merlin 
firmware provides. The instructions here should work on other Asus routers with Meriln firmware but might need some modifications to work on DDWRT or Tomato.

For pulling and persisting the metrics I am be using [Prometheus](https://prometheus.io) which is an open-source monitoring solution. I have it installed
on a home server which is up 24x7 and connected to the router. A static IP is configured for the server which makes things convenient but the setup
will also work with a dynamic IP address assignment. [Grafana](https://grafana.com/) is used to create a dashboard for router health. 

We will be installing `node_exporter` on the router which will provide us a lot of metrics out of which following are of interest:

1. CPU utilization
2. Memory usage
3. Network activity
4. Disk usage

At the end of this post our dashboard will look something like this:

TODO: Add picture of final dashboard here

## Prerequisites

You need an ARM based router with minimum 256MB of RAM. In addition to this you will need sufficient disk space (100MB) for exporters and configurations.
At the time of this writing `node_exporter` is using between 1-3MB of memory with router's total memory utilization at 63%. The size of exporter binary 
is 16.5MB. You also need to be able to SSH into the router.

You need to enable Custom JFFS scripts from `Administration > System` in your WebUI.

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
|- home
  |- ashwini
|- tmp
</pre>

TODO: add command for creating directory structure

## Installing node_exporter

Hop on to the [releases page](https://github.com/prometheus/node_exporter/releases) and grab the latest arm build for the router. For AC68U we need the ARM5
build. I am grabbing the version 1.0.0.

You can get to your router terminal via SSH and run following commands to get the exporter.  **(Change \<version\> with the version you want.)**

{% highlight bash %}
cd /mnt/<disk>/home/ashwini
wget https://github.com/prometheus/node_exporter/releases/download/v1.0.0/node_exporter-<version>.linux-armv5.tar.gz
tar xzf node_exporter-<version>.linux-armv5.tar.gz
mv node_exporter-<version>.linux-armv5/node_exporter /mnt/<disk>/bin/node_exporter
rm -rf node_exporter-<version>.linux-armv5*
{% endhighlight %}

Check if `node_exporter` is working using this command. It should print out the version and build information on the console. If you get an error you probably 
grabbed a build for wrong platform.

Once the exporter is validated we need to configure the node_exporter to start on boot. First we create a service file `/mnt/<disk>/etc/services/node_exporter.service`
with following content in it. This scripts checks if the exporter is already running by fetching the PID of `node_exporter`. If not running it will start the node_exporter
in background.

{% highlight bash %}
#!/bin/sh

ROOT_DIR=$1
CMD="$1/bin/node_exporter"
LOG="$1/var/log/node_exporter.log"

pidof node_exporter

if [[ $? -ne 0 ]] ; then
    $CMD --web.listen-address=":9101" >> "$LOG" 2>&1 &
else
    echo "Already running"
fi
{% endhighlight %}

Mark this file as executable:

{% highlight bash %}
chmod 755 /mnt/<disk>/etc/services/node_exporter.service
{% endhighlight %}

Now we can run the service to start our exporter.

{% highlight bash %}
/mnt/<disk>/etc/services/node_exporter.service
{% endhighlight %}

Visit `http://<your_router_ip>:9101/metrics`. If you are greeted with a page with prometheus metrics we are good to go. Following
is a snippet of what you should see on the page.

<pre>
# HELP go_gc_duration_seconds A summary of the pause duration of garbage collection cycles.
# TYPE go_gc_duration_seconds summary
go_gc_duration_seconds{quantile="0"} 0.000138551
go_gc_duration_seconds{quantile="0.25"} 0.00018608
go_gc_duration_seconds{quantile="0.5"} 0.000246878
go_gc_duration_seconds{quantile="0.75"} 0.000339976
go_gc_duration_seconds{quantile="1"} 0.010150014
go_gc_duration_seconds_sum 2.515875774
go_gc_duration_seconds_count 3756
--- truncated ---
</pre>

Now we configure this service file to be run on boot. During the boot process we need to ensure that the external usb is mounted
before we call this script. We will be using the `post-mount` script. Create a file `/jffs/scripts/post-mount` if it not already 
exists. Add following content to the file. The `post-mount` script is passed the mountpoint as the first argument. In 
the script we check if the disk mounted is the one which has our service file. If yes then we invoke the script. Beware that you
should not execute any blocking action in this script as the boot process waits for this script to exit. (**Change \<disk\> with correct disk name**)

{% highlight bash %}
#!/bin/sh

MOUNTPOINT=$1

if [[ $MOUNTPOINT == "/tmp/mnt/<disk>" ]]; then
    echo "$1 Mounted" >> "$1/var/log/syslog"
    CMD="$1/etc/services/node_exporter.service"
    $CMD "$1"
fi
{% endhighlight %}

Thats it for the setup on the router end. Now we move on to configure Prometheus.

## Configuring Prometheus
