---
layout: post
title: SaltStack - A better salt/top.sls - Part 2
modified:
categories: 
description:
tags: []
image:
  feature:
  credit:
  creditlink:
comments:
share:
date: 2015-11-19T17:33:33-08:00
---

## Dynamic goodness into the top.sls state file

### In search of a a better top.sls file. 

In [Part 1](/saltstack/salt-better-top-state-file/) we started getting the salt/top.sls file into a
more dynamic setup.  In this next part, we will continue working on it to make it more generic.

We will now pull in the data for the role states from a pillar node that matches the role.
<!--more-->

So, here is what we have from the last article :
{% highlight salt linenos %}
{% raw %}
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
{% endraw %}
{% endhighlight %}

Our static roles grain will be parsed, and whatever entries are there will be converted to a salt state
call.  We now need to change it so that the role entry does not need to match directly to a state, but
instead will be used to pull a collection of states from pillar data into the top.sls

Lets work with the 'monitor' server name this time, and convert that into a role.  There is no salt state
called 'monitor', just the grafana and shinken states (I have salt formulas setup for grafana and shinken).

Here is the pillar data I will be using to feed in the states.
{% highlight salt linenos %}
{% raw %}
#!yamlex
include:
  - grafana
  - shinken

states: !aggregate
  monitor: !aggregate
    - grafana
    - shinken
{% endraw %}
{% endhighlight %}

I better explain a bit about the contents.  
First, this file is in /srv/saltstack/pillar/roles/monitor/init.sls on my installation.  My pillar_root
is set to /srv/saltstack/pillar.  This is not standard, but /srv is not dedicated to salt, and /srv/pillar 
is not contained enough for my likes. 

- _Line 1_ : I use Yamlex as my yaml processor for my salt files.  It allows me to be able to do a
deep merge with my yaml data so I can override/append to my structure.  I get this with Hiera in Puppet
and I have grown used to being able to have that functionality.  I will make a blog post on it.
- _Line 2-4_ : I bring in /srv/saltstack/pillar/grafana/init.sls and /srv/saltstack/pillar/shinken/init.sls
These contain pillar data that is used to configure those states.
- _Line 6-9_ : This is where I define the array to store the states I want to bring into the salt/top.sls file.  The 
pillar lookup path for these will be [states.monitor] 

### Adding code to top.sls to bring in the pillar data 

{% highlight salt linenos %}
{% raw %}
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - shinken
  {% set roles = salt['grains.get']('roles',[]) -%}
  {% for role in roles -%}
  {% set states = salt['pillar.get']('states:'+role,[]) -%}
  {% if states -%}
  'roles:{{ role }}':
    - match: grain
    {% for state in states -%}
    - {{ state }}
    {% endfor -%}
  {% endif -%}
  {% endfor -%}
{% endraw %}
{% endhighlight %}

I have changed up the 'code' section for the roles.

- _Line 7-8_   : No change from before
- _Line 9_     : We use pillar.get all to bring in the pillar data 'states:ROLE' where ROLE is 
replace with the variable from the for loop.
- _Line 10_    : If there the list is empty, do nothing
- _Line 11-12_ : Build the normal top.sls notation to match a grain
- _Line 13-15_ : Loop through all the states listed in the pillar data and place them down

### Test on the minion to see if it works

In the previous article I was using a minion with the grain roles set to be 'influxdb' and 'python'.  The minion
we are working with this time needs to have the roles grain set to just 'monitor'.  

_/etc/salt/grains_
{% highlight salt linenos %}
{% raw %}
roles:
  - monitor

{% endraw %}
{% endhighlight %}

``salt-call --log-level=debug state.show_highstate``

A few lines down in the spam of debug logging (while we are learning, the more verbose the better) :
{% highlight salt linenos %}
{% raw %}
....
[DEBUG   ] LazyLoaded grains.get
[DEBUG   ] LazyLoaded pillar.get
[DEBUG   ] Rendered data from file: /var/cache/salt/minion/files/base/top.sls:
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - shinken

[DEBUG   ] LazyLoaded config.get
[DEBUG   ] Results of YAML rendering: 
....
{% endraw %}
{% endhighlight %}

Humm.  Well, that didn't work.  I expected to see a ``roles:monitor`` entry at the end.

Why don't I ?  

Well, we have not changed the Pillar top.sls!  It also needs some of this magic
so that it includes the pillar data that we setup for the states.  Below I will do that.

### Back to the master to fix pillar/top.sls

{% highlight salt linenos %}
{% raw %}
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - monitoring
  {% set roles = salt['grains.get']('roles',[]) -%}                                                                     
  {% for role in roles -%} 
  'roles:{{ role }}':
    - match: grain
    - roles.{{ role }}
  {% endfor -%} 
{% endraw %}
{% endhighlight %}

Remember, this is the pillar/top.sls - It is _NOT_ the salt/top.sls.  

Similar to salt/top.sls, the pillar/top.sls brings in the other files under pillar/ as directed.
We will use the same kind of loop iteration that we se for the salt/top.sls.  

We load up the grains , and loop through to build what we want.

I have not investigated a way to show the raw generated pillar/top.sls that we can see in the debug
output from show_highstate.  I will just use the pillar.items call to see if our data has made its way.

``salt-call --log-level=debug pillar.items states --out=json``

{% highlight json %}
{% raw %}
{
    "local": {
        "states": {
            "monitor": [
                "grafana", 
                "shinken"
            ]
        }
    }
}
{% endraw %}
{% endhighlight %}

There it is.

The raw generated pillar/top.sls would have looked something like this if you could see it :
{% highlight salt %}
{% raw %}
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - monitoring
  'roles:monitor':
    - match: grain
    - roles.monitor
{% endraw %}
{% endhighlight %}

This would cause the ``/srv/pillar/roles/monitor/init.sls`` file to be loaded in to the pillar data.

From the top of the article, we can see where we added the states.monitor list into this file.

Now when the salt/top.sls is generated, it will loop through the listed states under each role key.

Try the salt.show_highstate call again

### Test on the minion to see if it works

``salt-call --log-level=debug state.show_highstate``

A few lines down again...
{% highlight salt %}
{% raw %}
....
[DEBUG   ] LazyLoaded grains.get
[DEBUG   ] LazyLoaded pillar.get
[DEBUG   ] Rendered data from file: /var/cache/salt/minion/files/base/top.sls:
base:
  '*':
    - default
  '*monitor*':
    - grafana
    - shinken
  'roles:monitor':
    - match: grain
    - grafana
    - shinken
    

[DEBUG   ] LazyLoaded config.get
[DEBUG   ] Results of YAML rendering: 
....
{% endraw %}
{% endhighlight %}

todo : Insert success meme here ! :)

Remember, you are seeing the 'generated' salt/top.sls, it will be different for every minion based on
the roles grain that is set.  If we ran it on the metrics server we were building in the last article
it would only show the roles:influxdb and roles:python sections.  In fact we now need to go back and setup
a different role for it, and setup the pillar data for the states we want (influxdb and python).

Now we can go back and delete the ``'*monitor*':`` entry from the top.sls since we don't need it
anymore.  The proper states are brought in dynamically.

So, our final salt/top.sls looks like this:
 
{% highlight salt %}
{% raw %}
base:
  '*':
    - default
  {% set roles = salt['grains.get']('roles',[]) -%}
  {% for role in roles -%}
  {% set states = salt['pillar.get']('states:'+role,[]) -%}
  {% if states -%}
  'roles:{{ role }}':
    - match: grain
    {% for state in states -%}
    - {{ state }}
    {% endfor -%}
  {% endif -%}
  {% endfor -%}
{% endraw %}
{% endhighlight %}

and our pillar/top.sls looks like:

{% highlight salt %}
{% raw %}
base:                                                                                                                   
  '*':
    - default
  {% set roles = salt['grains.get']('roles',[]) -%} 
  {% for role in roles -%} 
  'roles:{{ role }}':
    - match: grain
    - roles.{{ role }}
  {% endfor -%} 
{% endraw %}
{% endhighlight %}

More /srv/saltstack/pillar/roles/{ROLENAME}/init.sls files will need be be created as I make more roles.

We now have top.sls files that we don't need to touch to be able to handle all kinds of different states or 
pillar data.  We are not changing the 'code' just the data in the pillar.


