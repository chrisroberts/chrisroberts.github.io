---
title: "Hashes == 42, a love story"
date: 2014-11-20 07:45:33 -0700
tags: [ruby, hashes, configuration, answer]
---

Thoughts on application configuration and snowflake implementations.

## Hashes == 42 (how I came to love hashes)

Hashes have always just been another data type to me. Something
to be used within a proper model. As I've been working more and
more with automation, and more importantly, configuration automation,
I have grown weary of applications defining their own configuration
conventions. I fully understand the want to have a pure configuration
model. Implemented with love and care to do exactly the things
it's supposed to, without the cruft of features that may never
be utilized. The want to make the implementation beautiful. Yet
that beauty can be quickly lost once someone actually tries to
use it.

### Configuration hashes

JSON is awesome. Really, it's like a wheel. Simple, effective and
compatible pretty much everywhere. No matter what language I use,
there will likely be an existing JSON parser ready for me to
utilize. This fact is important not just for the application
begin configured, but also for the application that is providing
the configuration.

Many times configuration is viewed as a person flipping flags
or defining static values. As an application developer, this is
not a "wrong" view, just a narrowed view. Once the application
breaks out to a scaled deployment, no person will be defining
these configurations. It's this point that being able to
define a configuration from a Hash becomes important.

### Automation

I mainly use Chef, so we'll just use that frame of reference
for these exmaples. However, this spans any automation framework
that allows for providing complex logic.

If I write an application that uses a customized configuration
format it will require customized logic for providing that
configuration. At the surface, this seems like a pretty small
feat. A `template` can be defined so we can programmatically
build the configuration. But, can the configuration be easily
built programmatically? One of my favorite examples of this
is the configuration file for an LXC instance.

```
lxc.network.ipv4 = 127.0.0.2
lxc.network.ipv4.gateway = 127.0.0.1
```

This small snippet shows how a simple configuration file
can grow in compexity very quickly. How can this be easily
defined with basic collection types (Hash/Array)? Ideally,
we would like to be able to provide some sane defaults which
the user (the automation system in this case) can override.

But basic types won't cover this type of configuration. Hashes
will end up stomping keys, and Arrays will end up needing custom
logic on "how" to process them. Especially since we can see how
Arrays may already need to be used when multiple values are provided
for a single "key":

```
lxc.cgroup.devices.allow = c *:* m
lxc.cgroup.devices.allow = b *:* m
```

The more we look at this configuration and its structure, the more
places we find customized logic is required and writing a simple
configuration file has now become a complete complex task.

### What if JSON sucks

Everyone has their love and hate for everything. But the important
part is the "serialization". Have a strong love for YAML because
you can't use ERB in JSON? While that may open another can of worms,
it's still preferable to some customized configuration library. Since
producing YAML or JSON is just as easy as consuming.

In the end, just be kind to your users. And not just the users running
your application on a one off machine, but also the operators attempting
to deploy them at scale.
