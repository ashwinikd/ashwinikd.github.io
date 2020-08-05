---
layout: post
title:  "Monitoring WAN status on your wireless router"
# date:   2020-07-30 16:14:42 +0530
categories: monitoring router prometheus
series: "router-monitoring"
---

This is fourth in [series of posts]({{ site.url }}/series/router-monitoring) on monitoring your wireless router. In this post we will setup 
[blackbox exporter](https://github.com/prometheus/blackbox_exporter) on the router which will allow us to check the status of internet connection. I have an 
[Asus RT-AC68U](https://www.asus.com/in/Networking/RTAC68U/) with [AsusWRT-Merlin](https://www.asuswrt-merlin.net/) firmware installed on it. 
This allows me to setup cron jobs and custom scripts on the router which is required to setup the monitoring. You can check out the complete 
list of [additional features](https://www.asuswrt-merlin.net/features) that Merlin firmware provides. The instructions here should work on 
other Asus routers with Meriln firmware but might need some modifications to work on DDWRT or Tomato.

For pulling and persisting the metrics I am be using [Prometheus](https://prometheus.io) which is an open-source monitoring solution. I have it installed
on a home server which is up 24x7 and connected to the router. A static IP is configured for the server which makes things convenient but the setup
will also work with a dynamic IP address assignment.

## Prerequisites

You need an ARM based router with minimum 256MB of RAM. In addition to this you will need sufficient disk space (100MB) for exporters and configurations.
The size of exporter binary is 14.5MB. You also need to be able to SSH into the router.

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
  |- user
|- tmp
</pre>

You can create the directory structure using following command. **(Change \<disk\> to the name of your disk.)**

{% highlight bash %}
mkdir -p /mnt/<disk>/{bin,etc/services,home/user,var/{log,run},tmp}
{% endhighlight %}

## Installing blackbox exporter

Goto the [release page of blackbox exporter](https://github.com/prometheus/blackbox_exporter/releases) and grab the latest release available. For Asus RT-AC series
we need the ARM5 build. I am grabbing version `0.16.0`. You can use following commands on router to get the release and set it up.

{% highlight bash %}
cd /mnt/<disk>/home/user
wget https://github.com/prometheus/blackbox_exporter/releases/download/v0.16.0/blackbox_exporter-0.16.0.linux-armv5.tar.gz
tar xzf blackbox_exporter-0.16.0.linux-armv5.tar.gz
mv blackbox_exporter-0.16.0.linux-armv5/blackbox_exporter /mnt/<disk>/bin/blackbox_exporter
rm -rf blackbox_exporter-0.16.0.linux-armv5*
{% endhighlight %}

Check if `blacbox exporter` is working by invoking the binary. It should print out the version and build information on the console. If you get an error you probably 
grabbed a build for wrong platform.

{% highlight bash %}
/mnt/<disk>/bin/blackbox_exporter --version
{% endhighlight %}

Once the exporter is validated we need to configure the blackbox exporter to start on boot. First we create a service file `/mnt/<disk>/etc/services/blackbox_exporter.service`
with following content in it. This scripts checks if the exporter is already running by fetching the PID of `blackbox_exporter`. If not running it will start the exporter
in background.

{% highlight bash %}
#!/bin/sh

ROOT_DIR=$1
CMD="$1/bin/blackbox_exporter"
LOG="$1/var/log/blackbox_exporter.log"

pidof blackbox_exporter

if [[ $? -ne 0 ]] ; then
    $CMD --config.file="$1/etc/blackbox.yml" >> "$LOG" 2>&1 &
else
    echo "Already running"
fi
{% endhighlight %}

Mark this file as executable:

{% highlight bash %}
chmod 755 /mnt/<disk>/etc/services/blackbox_exporter.service
{% endhighlight %}

Create a config file `/mnt/<disk>/etc/blackbox.yml` with following content.

{% highlight yaml %}
modules:
  http_2xx:
    prober: http
  http_post_2xx:
    prober: http
    http:
      method: POST
  tcp_connect:
    prober: tcp
  pop3s_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^+OK"
      tls: true
      tls_config:
        insecure_skip_verify: false
  ssh_banner:
    prober: tcp
    tcp:
      query_response:
      - expect: "^SSH-2.0-"
  irc_banner:
    prober: tcp
    tcp:
      query_response:
      - send: "NICK prober"
      - send: "USER prober prober prober :prober"
      - expect: "PING :([^ ]+)"
        send: "PONG ${1}"
      - expect: "^:[^ ]+ 001"
  icmp:
    prober: icmp
  dns_rp_mx:
    prober: dns
    dns:
      query_name: "robustperception.io"
      query_type: "MX"
      validate_answer_rrs:
        fail_if_not_matches_regexp:
         - "robustperception.io.\t.*\tIN\tMX\t.*google.*"

{% endhighlight %}

Now we can run the service to start our exporter.

{% highlight bash %}
/mnt/<disk>/etc/services/blackbox_exporter.service
{% endhighlight %}

You can check if blackbox exporter is running using command `pidof blackbox_exporter`. Once running you can visit 
`http://<your_router_ip>:9115/probe?type=dns&target=ashwinikd.github.io` to check if you get the metrics. Following
is a snippet of what you should see:

<pre>
# HELP probe_dns_lookup_time_seconds Returns the time taken for probe dns lookup in seconds
# TYPE probe_dns_lookup_time_seconds gauge
probe_dns_lookup_time_seconds 0.252522686
# HELP probe_duration_seconds Returns how long the probe took to complete in seconds
# TYPE probe_duration_seconds gauge
probe_duration_seconds 2.303525591
# HELP probe_failed_due_to_regex Indicates if probe failed due to regex
# TYPE probe_failed_due_to_regex gauge
probe_failed_due_to_regex 0
<strong>--- truncated --- </strong>
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
    CMD="$1/etc/services/blackbox_exporter.service"
    $CMD "$1"
fi
{% endhighlight %}

Thats it for the setup on the router end. Now we move on to configure Prometheus.

## Configuring prometheus

Add following job to your `scrape_configs` section and restart prometheus.

{% highlight yaml %}
  - job_name: 'wan_check'
    scrape_timeout: 2s
    metrics_path: /probe
    params:
      module: [tcp_connect]
    static_configs:
      - targets:
        - "8.8.4.4:53"  # Test various public DNS providers are working.
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <your_router_ip>:9115
{% endhighlight %}

### Bonus: Check DNS servers

You can also add following job to check the uptime of DNS servers you are using. I am using NordVPN DNS servers so you might have to change
this to the IP addresses of the servers you are using.

{% highlight yaml %}
  - job_name: 'wan_dns'
    metrics_path: /probe
    params:
      module: [dns_rp_mx]
    static_configs:
      - targets:
        # Change these IPs to the DNS servers you are using
        - 103.86.96.100
        - 103.86.99.100
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: <your_router_ip>:9115
{% endhighlight %}

## Prometheus queries

In this section we go over the queries for most common metrics that you might want to track.

### Internet uptime over last 30 days

This query gives you the number of minutes your internet connection was working over last 30 days. Note the `15 / 60` in the query. This
is to adjust for the 15 seconds scrape interval. You should change `15` to your scrape interval in seconds.

{% highlight promql %}
sum_over_time(probe_success{job="wan_check",instance="8.8.4.4:53"}[30d]) * 15 / 60
{% endhighlight %}

Use this query to check the number of seconds the blackbox exporter was running:

{% highlight promql %}
sum_over_time(up{job="wan_check",instance="8.8.4.4:53"}[30d]) * 15 / 60
{% endhighlight %}

If we divide the above two metrics `<internet_uptime> / <exporter_uptime>` we get the percentage of availability for the internet. Since my
scrape intervals are same for both tasks i am dropping the `15 / 60` component from both queries.

{% highlight promql %}
sum_over_time(probe_success{job="wan_check",instance="8.8.4.4:53"}[30d]) / sum_over_time(up{job="wan_check",instance="8.8.4.4:53"}[30d])
{% endhighlight %}

### Number of DNS servers working

Following query will give you the number of DNS servers that are working at any given monment:

{% highlight promql %}
sum(probe_success{job="wan_dns"})
{% endhighlight %}


