---
title: SaltStack - A better salt/top.sls
layout: post
categories: saltstack
description: Spreading some dynamic goodness into the top.sls state file
tags: [saltstack pillar jinja grains myway]
comments: false
date: 2015-11-18T19:28:01-05:00
---

# Spreading some dynamic goodness into the top.sls state file
## In search of a a better top.sls file. 

I have my code in all my states in separate formulas each with a separate file_root, and /salt/* files.  
I have all my data in a separate pillar layout.  Things are working good.

Now, I want to clean up the top.sls file so that I don't have to keep adding hostnames, and assigning the 'code' 
package names to them.  This is a job for the grains, and some template/code work.  At least I think it is.
Let's dive in and see if I can get it doing what I want.

An example of my current top.sls file (for me that is /srv/saltstack/salt/top.sls - a non-standard path [^1])
{% highlight yaml linenos %}
base:
  '*':
    - default
  '*influxdb*':
    - influxdb
    - python
  '*monitor*':
    - grafana
    - shinken
{% endhighlight %}

So, if the hostname has 'influxdb' in it, the salt state file from my influxdb formula gets 
called (starting by bringing in /srv/saltstack/formulas/influxdb-formula/influxdb/init.sls).  
If if the hostname has monitor in it, then likewise, grafana and shinken come into play.

What I want to be able to do is set some grains, and have this all changed up dynamically instead
of being based on host names.  Even better if the list of code that gets shoved in is actally pulled from
pillar data somewhere !

Let's start with the grain setup.  Bit of a chicken and egg problem since we need to have the grains setup 
on the minion before we run salt.  Down the road, other systems could be used to inject the 
grains (cloud-init, etc.) but for now we will just manually set them up.  This would be on the fresh 
instance, after salt-minion is installed, but before the first salt-call.

### On the minion
I am going to put them in /etc/salt/grains instead of the salt minion main config file (/etc/salt/minion).  
That file is placed there by the your distro packages, and has sane defaults in it.  I want to leave that
as is, and override changes either in /etc/salt/minion.d/* files, or in this case, the purpose built 
/etc/salt/grains yaml file.

{% highlight yaml linenos %}
roles:
  - influxdb
  - python
{% endhighlight %}

This will build a grain 'roles' that has an array of values.  I want to be able to have a few roles so 
I am going to start right out to the gate with an array, and make that work.  

Let's see if that worked (from the 

{% highlight yaml %}
# salt-call grains.items
....trimmed for size.....
    pythonversion:
        - 2
        - 7
        - 9
        - final
        - 0
    roles:
        - influxdb
        - python
    saltpath:
        /usr/lib/python2.7/dist-packages/salt
    saltversion:
        2015.8.1
    saltversioninfo:
        - 2015
        - 8
        - 1
        - 0
....trimmed for size.....

{% endhighlight %}

There they are along with all the other default caculated grains.

Now, onto getting top.sls file to act on it.

### On the master

So, does it work for targeting a minion ?
{% highlight bash %}
# salt --log-level=debug -C 'G@roles:influxdb' test.ping
kt-testminion-1.kerkhofftech.ca:
    True

{% endhighlight %}

So far, so good.  Now onto iterating it for config.

What we want the top.sls to look like (for the role handling) is something like this :
{% highlight yaml %}
  'roles:influxdb':
    - match: grain
    - influxdb
{% endhighlight %}

But we don't want to have to write this over and over - I don't want to have to touch the top.sls 
if I can avoid it - That is part of my code, not my config.

We're going to use some jinja goodness to build what we want.

{% highlight yaml linenos %}
  {% set roles = salt['grains.get']('roles',[]) -%}
  {% for role in roles -%}
  'roles:{{ role }}':
    - match: grain
    - {{ role }}
  {% endfor -%}
{% endhighlight %}

Broken down we have:
* Line 1 : Use the salt function to get the 'roles' grain, and default to an empty arry if not there.
* Line 2 : For each role in the roles array : [influxdb, python] in our case
* Line 3 : output ``roles:influxdb`` or ``roles:python`` , again using our sample grain data
* Line 4 : output ``influxdb`` or ``python`` so that when it matches that formula init.sls gets pulled in.
* Line 5 : end the for loop

So, this is my full top.sls now
```
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - shinken
  {% set roles = salt['grains.get']('roles',[]) -%}
  {% for role in roles -%}
  'roles:{{ role }}':
    - match: grain
    - {{ role }}
  {% endfor -%}

```


Give it a try - **From the minion now**, :
```salt-call --log-level=debug state.show_highstate```

You should see a massive load of debug data , but scroll through it, near the top, and you will see your top.sls
file 

```
[DEBUG   ] Rendered data from file: /var/cache/salt/minion/files/base/top.sls:
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - shinken
  'roles:influxdb':
    - match: grain
    - influxdb
  'roles:python':
    - match: grain
    - python
```

todo : Insert success meme here ! :)

Part 2 - What if the role doesn't match directly with a single formula I want included ?  What if I want
multiple formulas to be stuffed in here ? To the Pillar Batman!


[^1]: I hate how the standard Saltstack locations crap all over my /srv directory as if it owns it.  ie. /srv/formulas is a silly place for Salt Formulas.

