---
title: "Flocking Chefs"
date: 2012-12-09 08:24
tags: [chef, communcation, internode, flock_of_chefs]
---

Internode communication via flocks of chefs.

## Flock of Chefs

When building and configuring nodes (assuming with Chef) one issue
that will commonly come up is the requirements of remote states for
nodes to properly converge. There have been many different approaches
to solving this issue. Searching for node attributes signalling a
desired state has been reached is one of the most common approaches. This
approach, however, can be slow and requires custom logic built into
recipes around success and failure states of the search.

Instead of searching for nodes that have reached a specific state via
node attributes on the Chef server, what if nodes could talk to each
other? This would allow nodes to not only query directly for state
on a remote node but would also allow for nodes to notify remote nodes
when a desired state has been reached. Flock of Chefs aims to provide
this functionality plus a few extras. Lets take a closer look.

### Tooling

The tooling used to back flock building:

* DCell: https://github.com/celluloid/dcell
* Zookeeper: http://zookeeper.apache.org/

### Requirements for node setup

The flock_of_chefs cookbook uses the zookeeperd cookbook for auto
zk node discovery. The configuration and setup of zookeeper nodes
and clusters can range wildly, so the flock_of_chefs cookbook will
not set any up or attempt configuration. For testing though, just
adding the zookeeperd cookbook into the run list of the test nodes
will allow a cluster to setup for the flock to use.

```ruby
# role/flocked.rb
name 'flocked'
run_list(%w(
  recipe[zookeeperd::server]
  recipe[flock_of_chefs]
))
```

### New features

The Flock of Chefs library will provide two new primatives to Chef. First,
lets look at how to use the primatives within a recipe, and then look at
what these new primatives are doing to Chef under the hood.

* remote_notifies

This is used to send notifications to remote nodes. It works much the same
as the regular `notifies` within Chef, with the extra ability to send the
notifications to other Chef nodes. In practice, a recipe would look like
this:

```ruby
resource '/tmp/notify_test' do
  content 'test content'
  remote_notifies :create, 'file[remote_notify_test]', :node => 'remote.1'
end
```

* remote_subscribes

This is the inverse of `remote_notifies` and works much the same as the
regular `subscribes` within Chef. It will search for the nodes provided
and register a notify with them, so resources can subscribe to changes
occurring on resources within other nodes. In practice, a recipe would
look like this:

```ruby
resource '/tmp/subscribe_test' do
  content 'test content'
  remote_subscribes :create, 'file[/tmp/notify_test]', :node => 'remote.2'
end
```

For simplitity sake we are using explicit node names for our subscriptions
and notifications. Support is available for using search to determine applicable
nodes, but we can get to that later.
