---
layout: post
title: Collectd, the long way, to Grafana - Part 1
subtitle: Tags and Fields
date: 2015-12-13T19:56:07-08:00
modified:
categories: devops metrics
description:
tags: [influxdb shinken collectd grafana]
comments: true
share: true
---

# Tags and Fields and Shinken oh my..

(todo: insert funny 'trigger' warning image)

### What is the problem ?
I have my flow working.  Data is generated in Collectd, and sent to my monitoring server
where a [Shinken](http://www.shinken-monitoring.org/) module [mod_collectd](http://shinken.io/package/mod-collectd)
receives it.  It bounces around Shinken for some actual monitoring work, and then the metrics get shipped off
to a [InfluxDB](https://influxdata.com/) server for eventual visualization in [Grafana](http://grafana.org/).
The whole thing is deployed and configured using Saltstack with feedback as various minions come online.
That is a separate series for the future :).. 

To the problem - Everything was flowing fine, but now I want to start setting up some Grafana views of the flood
of metrics coming in.  A few test ones on __Users__ or __Load__ are all fine, but when I hit the __CPU__ metrics we have
a bit of a problem....

<!--more-->

Here are some of my measurements in InfluxDB

{% highlight console %}
SHOW MEASUREMENTS

metric_cpu_0_idle
metric_cpu_0_nice
metric_cpu_0_softirq
metric_cpu_0_steal
metric_cpu_0_system
metric_cpu_0_user
metric_cpu_0_wait
metric_cpu_1_idle
metric_cpu_1_nice
metric_cpu_1_softirq
... 
{% endhighlight %}
and on..

As you can see, I could get this into Grafana with one query for each.  For graphite backed data
it looks like Grafana could use the dot notation and with various filters create aliases based on 
the above.  

It looks like now with InfluxDB 0.9+, the proper way to go is with tags, and using the `$tag_sometagname` syntax
in the Grafana Alias By section.

* How does the CPU 0 Idle Metric from a particular host make it through the chain to wind up in the InfluxDB ?
* Where does it become the measurement name metric_cpu_0_idle ?
* How do I change it into 'cpu' measurement name (since that makes sense), with the tag `cpu=0` and `state=idle` ?

### Start the debugging

I am going to start at the begining, the line protocol from CollectD to Shinken.

We could start digging into the protocol documentation, etc. but let's start with just looking at the raw
line protocol since I am already in on the system and I don't want to switch to my browser to look up the protocol.

So, of to one of my server instances that has collectd -> shinken in progress.
Default port is UDP 25826

{% highlight console %}
root@abc-metrics-1:/etc/collectd# tcpdump -i eth0 -nA port 25826
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes
08:59:32.061094 IP 10.1.1.42.33460 > 10.1.1.32.25826: UDP, length 1335
08:59:32.061256 IP 10.1.1.42.33460 > 10.1.1.32.25826: UDP, length 1261
08:59:32.061595 IP 10.1.1.42.33460 > 10.1.1.32.25826: UDP, length 1319
{% endhighlight %}

Yup, data.  Now I need to see the actual data.


{% highlight console %}
`root@abc-metrics-1:/etc/collectd# tcpdump -i eth0 -nA udp port 25826
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on eth0, link-type EN10MB (Ethernet), capture size 262144 bytes


09:03:42.061503 IP 10.1.1.42.33460 > 10.1.1.32.25826: UDP, length 1309
E..9..@.@.a.
..*
.. ..d..%..... abc-metrics-1.yxx.example.ca............l.  ..............cpu.....0.....cpu.... wait..............%...........H.....softirq.............. j..........P....
steal..........................`.....1....  user..............X@..........e.... nice..........................h.....system..........................+.....0.....interrupt..........................le....1....  idle.............t............t.....interrupt..........................w.....softirq...............]..........{....
steal................................2....  user..............' ................system...............)..........._... idle.............u@&................interrupt...............................  nice................................softirq...............!................3....  nice...........................3....2.... wait..............1.................3.....system...........................:....2....
steal..........................q5....1....  wait................................3.... user...............z................softirq...............................  wait..............Iv...............
steal...........................#....4....  nice.........................../....system...............a................3.....interrupt................................4....  wait..............J.................interrupt................
09:03:42.112363 IP 10.1.1.42.33460 > 10.1.1.32.25826: UDP, length 1319
E..C..@.@.a.
..*
.. ..d../..... abc-metrics-1.yxx.example.ca............v.  ..............cpu.....4.....cpu.....softirq...........................H...
steal................................3....  idle.............u3k................4.............u..................5.....system...............................  nice...........................z... user...........................L....4..............3'................5....  idle.............gWu...........j....softirq...............h...........-...
users.........
users....................?................cpu.....5.....cpu.... wait...............<...........i...
steal................................interrupt..........................Gi... swap..........swap_io.....in..........................V
....out................................df.....run.....df_complex....  free...............A........./`.....df_inodes.... used..............G@................df_complex...............@........./x=....run-lock.....reserved........................./|....  used........................./4.....run.....df_inodes.....reserved...........................@... free.............D?A........./......run-lock.....reserved................................run.....df_complex........................./......run-shm....  free.............2.A........./s.....run-lock..............TA........./......run-shm.... used...............@
{% endhighlight %}


So, just a quick look the phrase 'metric_cpu_0_wait' is not there.  It looks like that the separate fields are indivitual variables in the protocol.

The next target will be the the Shinken `mod_collectd` module that is getting this data.  Off to the Python !

The module (on my install) is in `/var/lib/shinken/modules/mod_collectd/module.py`.   I love being able to just see the source right there, and tweek
it as needed and this applies for the entire Shinken system.  This is one reason I have switched to Shinken from Nagios.

Digging into the mod_collectd and having to grok a fair bit of python I see where the `Element.get_command()` method is puting it all together.
It is all getting joined together to make it fit with in the Shinken (and Nagios) perf data layout - `some_variable_name=value,next_variable=value,etc..`

`$type-$plugininstance-$typeinstance_$valueindex=$value`

For examples, 
CPU #4 Interrupt collectd metric becomes
`cpu-4-interrupt=0.0`

1, 5, and 10 minute load values become:
`load_0=0.95, load_1=0.98, load_2=0.78`

(the CollectD load plugin sends 3 values).

The df plugin turns into :
`df_complex-root-reserved=1734762496.0, df_complex-root-used=2364227584.0, df_complex-root-free=29587415040.0`

..because the actual CollectD type for the df plugin metric is called df_complex.

Also, I will note that the mod_collectd plugin also has special handling for any module that 
you label as a _Grouped Plugin_.  This will cause the metrics shipped back into Shinken to 
be under the service name `$plugin` instead of service name `$plugin_$plugin_instance` 
(from collectd data package)

I have `cpu,df,disk,interface,and ntpd` listed as _grouped plugins_ in the mod_collectd.conf file (in Shinken)

It looks like it is best to leave this module alone since we don't know what other components might be consuming the Perf Data that is getting injected.
This means that our next place to look at chaning things is in the mod_influxdb module itself.

`/var/lib/shinken/modules/mod_influxdb/*.py`

Note that I am currently using the `mod_influxdb` from its master branch on [Github](https://github.com/savoirfairelinux/mod-influxdb) since
there are bug fixes in it that deal with the fact that recent InfluxDB 0.9 subversions changed the API with respect to the timestamp value
and it now defaults to a 64Bit timestamp instead of 32Bit.  (if all your shinken->influxdb values are coming up as 1970, that is probably why)

`mod_influxdb` runs under the Broker Daemon, and is passed down information in the form of a 'brok'.
The Broker will call the module at the following points:

{% highlight python %}
manage_service_check_result_brok()
manage_host_check_result_brok()
manage_unknown_host_check_result_brok()
manage_unknown_service_check_result_brok()
manage_log_brok()
{% endhighlight %}


It is all done by the `basemodule.manage_brok()` function that will call on this pattern :

`'manage_' + brok.type + '_brok'`

Our perf data is coming into the module via `manage_service_check_result_brok`

Right at the start, we can see the tags getting setup :
{% highlight python %}

       tags = {
            "host_name": data['host_name'],
            "service_description": data['service_description']
        }

{% endhighlight %}
 
Sure enough, if you look at the raw InfluxDB data for metric_cpu_0_idle

{% highlight console %}
time                  host_name                     min service_description     unit     value
2015-12-12T05:03:12Z  "abc-metrics-1.yxx.example.ca"  0       "cpu"           "bytes/s"   3509470
2015-12-12T05:03:23Z  "abc-metrics-1.yxx.example.ca"  0       "cpu"           "bytes/s"   93.9
.....
{% endhighlight %}


You can see that the `host_name` and `service_description` tags are being set.  We want more..
_I_ want the metric to be called `cpu`, and the tags to be something like `instance=0` and `type=idle`.

It looks like this is the module we need to hack on to make this work.  It looks like a fair bit of work 
though.. CPU will be different from DF and Load.  We need to think of a layout that will work for all
modules, or most modules, and provide some way to act differently for certain exceptions.

Our pattern (small sample)
{% highlight console %}
cpu-4-interrupt
load_0
load_1
load_2
df_complex-root-reserved
df_complex-root-used
df_complex-root-free
{% endhighlight %}


Well, the metric name - easy .. Always is the first before the _

Ah, what about the users plugin ? We have not looked at that one yet.

`users=2.0`

So, there might not always be a `_`

The patterns are 
{% highlight console %}
$METRIC_$INSTANCE_$TYPE
$METRIC_$TYPE
$METRIC
{% endhighlight %}

I feel a regex coming....(todo: insert sad face)..

So I start pondering the regex, but I don't want to ponder the regex, so the ADHD kicks in and I start 
to ponder the _$INSTANCE_ vs _$TYPE_ that show up in my InfluxDB data.  
"Hold on!" says my brain, "I think I remember the load metric showing up as 
load_1, load_5, load_15, not load_1 to load_3.."

Quick check of Influx 'SHOW MEASUREMENTS', and sure enough, brain was correct.

WTH?  Wait a second, the 'minutes' setting is not comming in from CollectD data, just an array of 3 values.
Who/What is telling Shinken that it should be 1, 5, and 15 Minutes ?

### Side Trip !  Off to the Triggers !

Triggers are a unique feature of Shinken that helps make all the Collectd->Shinken->InfluxDB->Grafana thing work.
The documentation says that Triggers are 'not for production', but the documenation is old, and if what I need to make everything super cool is 
'not for production', then unless my Shinken is connected to someone's pacemaker, or a furnace boiler, I am going to use it!
Besides, it looks like the trigger part of the code hasn't been touched in almost a year - sounds stable enough for me ;)

(todo: insert picture of rebel)

Triggers are little bits of python code, that will be called as a 'result' comes in.  
They are defined at the object level.

Here is my CPU service object :

{% highlight console %}
define service{
   service_description  cpu
   hostgroup_name       collectd
   use                  collectd-generic-service
   register             1
   check_command        _echo
   trigger_name         collectd_cpu
}
{% endhighlight %}
The trigger is in `/etc/shinken/triggers.d/collectd_cpu.trig` .

It is python code.   
ps.. Some of these come from the collectd pack [here](https://github.com/shinken-monitoring/pack-collectd) and [here](https://github.com/savoirfairelinux/monitoring-tools)

The trigger also works like a normal nagios module, so at this point in the data flow, you can trigger the alerts for OK, Warning, Critical, etc.
Pretty cool eh? (todo: insert Miley Cyrus 'pretty cool')

Now, the documentation is pretty light on triggers (being beta and all), so we are going to have to dive 
into the code to see what things we can do.  There is some logging already setup to help us as well
(starting with [trigger_functions.py](https://github.com/naparuba/shinken/blob/master/shinken/trigger_functions.py)

Not sure what daemon it runs under, so I am going to kick up the `log_level=DEBUG` in the scheduler first as 
it seems like the right place.

Restart shinken, and start learning (grep'ing)...

Yup, scheduler it was.. Now the logs..

The data is coming out of our `mod_collectd` with basic performance data attached to it - the raw metrics.
The trigger is getting called and doing whatever it needs.  Our Load trigger code looks like this:

{% highlight python linenos %}
#!/usr/bin/env python
try:
    import logging
    backend = "Shinken"
    logger = logging.getLogger(backend)
    exit_code_output = {0: 'OK',
                        1: 'WARNING',
                        2: 'CRITICAL',
                        3: 'UNKNOWN',
                       }
    exit_code = 0
    # Get threshold
    data = {}
    warn = self.host.customs.get('_LOAD_WARN', None)
    if not warn is None:
        (data['warn_1'],
         data['warn_5'],
         data['warn_15']) = [float(x) for x in warn.split(",")]
    crit = self.host.customs.get('_LOAD_CRIT', None)
    if not crit is None:
        (data['crit_1'],
         data['crit_5'],
         data['crit_15']) = [float(x) for x in warn.split(",")]

    # Get perfs
    # TODO: Check why load-_0 and not load_0
    # Check the patch grouped_plugin in collectd_arbiter module

    error = False
    try:
        data['1'] = float(perf(self, 'load_0'))
        data['5'] = float(perf(self, 'load_1'))
        data['15'] = float(perf(self, 'load_2'))
    except ValueError:
        logger.error("A required perf_data is missing for %s" % self.get_full_name())
        logger.info("Dumping perf_data : %s" % self.perf_data)
        perf_data = ""
        output = "Error : A required perf_data is missing to compute trigger"
        exit_code = 3
        error = True

    if not error:
        # Prepare output
        if warn is not None and crit is not None:
            perf_data = ("load_1=%(1)0.2f;%(warn_1)0.2f;%(crit_1)0.2f;0 "
                     "load_5=%(5)0.2f;%(warn_5)0.2f;%(crit_5)0.2f;0 "
                     "load_15=%(15)0.2f;%(warn_15)0.2f;%(crit_15)0.2f;0" % data)
        else:
            perf_data = ("load_1=%(1)0.2f;;;0 "
                     "load_5=%(5)0.2f;;;0 "
                     "load_15=%(15)0.2f;;;0 " % data)
        output = "Load: %(1)0.2f,%(5)0.2f,%(15)0.2f" % data

        # Get status
        if warn is not None and crit is not None:
            for x in ['1', '5', '15']:
                if data[x] > data['crit_' + x] and exit_code < 2:
                    exit_code = 2
                    continue
                if data[x] > data['warn_' + x] and exit_code < 1:
                    exit_code = 1
                    continue

    # Finish output
    output = " - ".join((exit_code_output[exit_code], output))

    # Set ouput
    set_value(self, output, perf_data, exit_code)
except Exception, e:
    set_value(self, "UNKNOWN: Trigger error: " + str(e), "", 3)#              

{% endhighlight %}

It is pretty straight forward.  It collects some threshold values from the host object 
(self is filled with a few handy object collections), converts the `load_1`, `load_2`, 
and `load_3` into `load_1`, `load_5`, and `load_15` values, check them against the 
warning and critical thresholds (if set), then creates new per data in the generally 
accepted perf data format for Nagios (metricname=value;unitofmeasure;warnlevel;critlevel;min;max)
and then pushes new perf data back along with a new output message in general
for OK, Warning, Critical, Unknown.

### Back on track to the mod_influxdb module

This is not where we can change it up to add tags, but we can inject some other information 
into the perfdata.  Perhaps some kind of pattern hint for the mod_influxdb code to create tags?

I could just add things like tag_sometagname=somevalue here, and have mod_influxdb parse that, 
but what about things like the CPU perfdata that has a mess of different cpu numbers and types
all in one perdata collection.  tag_cpunumber=3 will not work for example, when the perf data 
has 8 different cpus in it.

Perhaps passing a 7th field after the 'max' that could be 'format' indicator for mod_influxdb.

Let's head back to the mod_influxdb code and see what we can do.  
Starting with the load metric, and then tackle the cpu metric.

`manage_service_check_result_brok` passes to a few functions that parse the brok data into data 
points for InfluxDB.

There is some good diagnostic logging, so I am going to increase the log_level=debug on the Broker daemon. (mod_influxdb is a broker module).

One handy line is the 'Generated Points' debug log.  Capturing that and running it through a 
python code formatter, this is the data that is generated for the load data brok :

{% highlight python %}
[{
    'fields': {
        'warning': 5.0,
        'critical': 5.0,
        'unit': u '',
        'value': 0.5,
        'min': 0.0
    },
    'time': 1450028858,
    'tags': {
        'service_description': u 'load',
        'host_name': u 'abc-metrics-1.yxx.example.ca'
    },
    'measurement': u 'metric_load_5'
}, {
    'fields': {
        'warning': 4.0,
        'critical': 4.0,
        'unit': u '',
        'value': 0.83,
        'min': 0.0
    },
    'time': 1450028858,
    'tags': {
        'service_description': u 'load',
        'host_name': u 'abc-metrics-1.yxx.example.ca'
    },
    'measurement': u 'metric_load_15'
}, {
    'fields': {
        'warning': 6.0,
        'critical': 6.0,
        'unit': u '',
        'value': 0.19,
        'min': 0.0
    },
    'time': 1450028858,
    'tags': {
        'service_description': u 'load',
        'host_name': u 'abc-metrics-1.yxx.example.ca'
    },
    'measurement': u 'metric_load_1'
}, {
    'fields': {
        'last_state_change': 1449980979.780328,
        'acknowledged': 0,
        'last_check': 1450028858,
        'state_type': 'HARD',
        'state': 0,
        'output': u 'OK - Load: 0.19,0.50,0.83'
    },
    'tags': {
        'service_description': u 'load',
        'host_name': u 'abc-metrics-1.yxx.example.ca'
    },
    'time': 1450028858,
    'measurement': 'SERVICE_STATE'
}]
{% endhighlight %}


So we can see the 3 variables in the perf data being converted into fields for Influx, and some Tags.
A the end of the collection, you can also see the service state is collected and pushed to InfluxDB
as well (very handy to mark point in time events overtop of the metric graphs)

For load, the 'service_description' is fine.  We just need to get 'measurement' to be just 'load' 
(or 'metric_load' - mod_influxdb is adding 'metric' to everything), and we need to get the 
'type' to be the 1,5,or 15..

Here is my diff to the function

{% highlight diff linenos %}
diff --git a/module/module.py b/module/module.py
index e7ec6cf..8ec4e34 100644
--- a/module/module.py
+++ b/module/module.py
@@ -111,14 +111,31 @@ class InfluxdbBroker(BaseModule):
                         value = float(value)
                     fields[mapping[1]] = value
 
+            # Create some extra tags for the type and instance in the e.name
+            computed_tags = tags.copy()
+            namesplit = e.name.split('_',3);
+            if len(namesplit)>2 :
+                #name_instance_type
+                computed_tags.update({'instance': namesplit[1]})
+                computed_tags.update({'type': namesplit[2]})
+            elif len(namesplit)>1 :
+                #name_type
+                computed_tags.update({'type': namesplit[1]})
+            if len(namesplit)>=1 :
+                logger.debug("[influxdb broker] Namesplit : %s" % namesplit)
+                logger.debug("[influxdb broker] Tags : %s" % computed_tags)
+                e.name = namesplit[0];
+
             if fields:
                 point = {
                     "measurement": 'metric_%s' % self.illegal_char.sub('_', e.name),
                     "time": timestamp,
                     "fields": fields,
-                    "tags": tags,
+                    "tags": computed_tags,
                 }
                 points.append(point)
 
         return points
{% endhighlight %}

There might be better ways to do this - More 'Pythonic', but it should be clear enough to understand what
we are doing.  I took a copy of the tags that are already passed into the function. Then I split up the 
current 'metric' name by the '_' that were added in by the mod_collectd module.  Then, based on how 
many are there, I fill out appropriate tags, and we create the point with the new created tags.

Result - Well, it works..

Let's see the measurements in InfluxDB now :
{% highlight console %}
SHOW MEASUREMENTS
name
---------
EVENT
HOST_STATE
SERVICE_STATE
metric_cached
metric_cpu
metric_free
metric_io
metric_load
.....
{% endhighlight %}

So, metric_cpu_#_???? is gone, now just metric_cpu..

Inside ?

{% highlight console %}
select * from metric_cpu limit 10

time                  host_name                     instance  min service_description type      unit            value
--------------------------------------------------------------------------------------------------------------------------
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "3"       0   "cpu"               "system"  "ops/s"         125922
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "3"       0   "cpu"               "user"    "s/s"           149497
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "3"       0   "cpu"               "idle"    "bytes/s"       16546393
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "3"       0   "cpu"               "nice"    "merged_ops/s"  0
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "3"       0   "cpu"               "softirq" "bytes/s"       4233
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "4"       0   "cpu"               "wait"    "ops/s"         477145
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "4"       0   "cpu"               "softirq" "bytes/s"       5206
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "5"       0   "cpu"               "wait"    "ops/s"         2075242
2015-12-13T18:55:08Z  "abc-metrics-1.yxx.example.ca" "3"       0   "cpu"               "steal"   "merged_ops/s"  0
...trimed....
{% endhighlight %}


Instance and type are filled out properly.  

Next is to setup Grafana to use these new tags in the alias.

