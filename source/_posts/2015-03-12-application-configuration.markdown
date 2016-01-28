---
title: "Application Configuration"
date: 2015-03-12 11:35:13 -0800
tags: [ruby, configuration, bogo]
---

Generic application configuration implementation within Ruby.

## Application Configuration

A few months ago I wrote a little bit about application configuration
and how annoying all the snowflake configuration libraries built into
applications makes actual usage. The main point being: configure your
application via a Hash data structure. Doing this means the ability
to provide configuration via JSON, YAML, XML or anything else that
can result in a Hash. It also means sanity when configuring applications,
especially when deployed.

So, after talking about not implementing snowflake configuration libraries
I decided to practice what I was preaching and write a reusable library
so I would stop recreating JSON compatible configuration support.

### bogo-config

The bogo-config library is a simple extension of providing application
configuration via Hash type. At its core, it simply reads a Hash from
a file, be it JSON, YAML, XML, or Ruby. It adds some extra sugar in
as well if you want to enforce some rules. Default usage is pretty
simple:

```ruby
require 'bogo-config'

config = Bogo::Config.new('/path/to/file.json')
p config.data
```

But we can also subclass and define configuration items:

```ruby
require 'bogo-config'

class MyConfig < Bogo::Config
  attribute :name, String, :default => 'unset'
  attribute :age, Integer, :coerce => lambd{|v| v.to_i}
end
```

This allows us to apply some rules to the configuration, and gives
us easy to use methods to access that data:

```ruby

config = MyConfig.new('/path/to/config.json')
p config.name
p config.age
p config.data
```

and data will hold all values included those not explicitly defined
via `attribute` entries. Easy!

* https://github.com/spox/bogo-config