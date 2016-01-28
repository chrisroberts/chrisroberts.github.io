---
title: "Cooking up partial run lists with Chef"
date: 2012-05-09 06:51
tags: [run_list, chef, recipes, runlist_modifiers, ruby, devops]
---

Modify run lists within chef.

## Modified Run Lists

Node convergence is an integral part of the Chef philosophy. The end goal of all
the cookbooks and recipes is reaching convergence on the node so Chef runs amount
to a no-op. This is a noble goal, and works well when cookbooks are built to always
"do the right thing" and services relied upon by cookbooks are always available. However,
there are many times when services may be down for maintenance, or a 3rd party cookbook
has a bug in a recipe which ends up raising an exception halting the Chef run.

In general, this may not be a problem. If a bug in a recipe is causing an error, fix
the bug and upload the modified cookbook. If a service is down, wait for the service
to come back online or modify the cookbook to not _rely_ on the service being available.
This is a fine solution in general, but it may not be applicable in all situations. In
many cases application deployments have been moved into Chef. Application updates are
deployed as the node converges, which is incredibly useful allowing a very tight
integration with the node configuration and the application. This tight integration
with Chef also introduces some strain.

Lets use an example situation. Chef is configured on a node to update its yum index
and then check the for the latest versions of some specific packages. In the majority
of Chef runs, convergence will be met without any updates to these packages. Now,
lets assume a new security patch needs to be applied to the application running on this
node and the application is deployed via Chef. The developers make their modifications,
run their tests and are ready for Chef to deploy the updated code. Chef runs and the
yum repositories happen to be down for maintenance causing an exception to be thrown. At this
point, the application now has no way of being deployed until either the recipe has
been modified, tested and uploaded or the service returns.

It's at this point the question arises, "why do we care about yum updates during an
application deployment?". From the Chef perspective they are important because
the node should always meet a point of convergence. From a developer perspective,
they don't care. The yum updates can happen later when the service becomes available and
the needed application update should just get out now. It is here that the approach differs
between the philosophical idea of what a chef run should do and the pragmatic approach
of getting done what is required right now.

## Modification Tools

### Run List Overrides

One tool available to allow this is the run list override. Available starting in the
0.10.10 version of Chef, a new command line argument is available: `--override-runlist`.
This option allows explicit specification of what is loaded in the run list of the
chef run, regardless of what is defined within the node information. This is highly
useful if a specific recipe is required  to run, or to remove a problem bit from the
run list of a node. One drawback to this approach is that it is an "all or nothing"
approach. If a recipe provided via a role is causing the problems, the only solution
is to remove the role entirely from the run list. This also means that any recipes defined
within that role, as well as any attributes, will no longer be available. The biggest
benefit of the override run list is the fact that it is built into Chef, and as such is
available anywhere the version is recent enough.

## runlist_modifiers Cookbook

This cookbook was born out of extra options for run list modification that were not
accepted upstream into Chef. This cookbook provides two extra tools for modifying a Chef run. One
allows providing a list of recipes that are restricted from running, the other allowing
a list or recipes that are the only recipes allowed within a runlist. First, we will
look at restricting recipes from running.

Example Recipes
---------------

First, lets define a couple very simple recipes in a test cookbook to demonstrate:

```ruby
# cookbooks/testbook/recipes/default.rb

include_recipe 'testbook::restricted'

ruby_block "loaded_default" do
  block do
    puts 'default'
  end
end
```

```ruby
# cookbooks/testbook/recipes/restricted.rb

ruby_block "loaded_restricted" do
  block do
    puts 'restricted'
  end
end
```

### Restricted Recipes

Restricted recipes have a very simple implementation. If a recipe is attempted to be
loaded that is within the restricted recipes list, an exception will be raised preventing
it and any recipes that depend on the restricted recipe from being loaded. This is a
very important point of the restricted recipes. When a recipe is defined within the
restricted recipe list, it and any recipes dependent on it are prevented from running.

Since this information must be found via attributes, the restricted recipes can be listed
via role. However, these are generally one off runs, so we will use a local json file
on the node.

```
# ~/attrs.json
{"restricted_recipes": ["testbook"]}
```

Now, chef-client is run with the json file provided:

* `chef-client -j ~/attrs.json`

With the restricted recipes enabled, warnings will now be produced when the recipe is
encountered and it will not be allowed to run:

```
[Wed, 09 May 2012 02:14:27 +0000] WARN: Restricted recipes modifier is enabled [testbook, testbook::default]
[Wed, 09 May 2012 02:14:28 +0000] WARN: Restricted recipe encountered: testbook -> Not Loaded
```

Restricted recipes that are dependencies of other recipes will make a note about the
dependency. Updated json file:

```
# ~/attrs.json
{"restricted_recipes": ["testbook::restricted"]}
```

And the run will now restrict both the default and restricted recipes from being run:

```
[Wed, 09 May 2012 02:22:09 +0000] WARN: Restricted recipes modifier is enabled [testbook::restricted]
[Wed, 09 May 2012 02:22:09 +0000] WARN: Restricted recipe encountered: testbook::restricted -> Not Loaded
[Wed, 09 May 2012 02:22:09 +0000] WARN: Restricted recipe dependency found. (testbook depends on testbook::restricted). testbook -> Not Loaded
```

### Allowed Recipes

Allowed recipes work slightly differently. Instead of allowing _only_ the allowed recipes
to run, the allowed recipes attribute is used to prune the run list. Any dependencies
required are also allowed to be loaded (unless they are specified within the restricted
run list). This becomes extremely useful if only a specific set of base recipes are desired
to run. A great example would be an application deployment recipe. All role information will
be loaded but the run list can be pruned to only contain the application deployment recipe.

Lets assume our run list looks like the following:

```
INFO: Run List expands to [users::sysadmins, sudo, chef-client::config, chef-client::cron, chef_gem, runlist_modifiers, testbook]
```

If we use the default recipe in testbook, we can easily see how dependencies (testbook::restricted) are allowed to load:

```
# ~/attr.json
{"allowed_recipes": ["testbook"]}
```

with the output:

```
[Wed, 09 May 2012 03:29:11 +0000] WARN: Allowed recipes modifier is enabled [testbook, testbook::default]
[Wed, 09 May 2012 03:29:11 +0000] WARN: Recipe encountered not found in allowed recipe set: users::sysadmins
[Wed, 09 May 2012 03:29:11 +0000] WARN: Restricted recipe encountered: users::sysadmins -> Not Loaded
[Wed, 09 May 2012 03:29:11 +0000] WARN: Recipe encountered not found in allowed recipe set: sudo
[Wed, 09 May 2012 03:29:11 +0000] WARN: Restricted recipe encountered: sudo -> Not Loaded
[Wed, 09 May 2012 03:29:11 +0000] WARN: Recipe encountered not found in allowed recipe set: chef-client::config
[Wed, 09 May 2012 03:29:11 +0000] WARN: Restricted recipe encountered: chef-client::config -> Not Loaded
[Wed, 09 May 2012 03:29:11 +0000] WARN: Recipe encountered not found in allowed recipe set: chef-client::cron
[Wed, 09 May 2012 03:29:11 +0000] WARN: Restricted recipe encountered: chef-client::cron -> Not Loaded
[Wed, 09 May 2012 03:29:11 +0000] WARN: Recipe encountered not found in allowed recipe set: chef_gem
[Wed, 09 May 2012 03:29:11 +0000] WARN: Restricted recipe encountered: chef_gem -> Not Loaded
[Wed, 09 May 2012 03:29:11 +0000] INFO: Processing ruby_block[loaded_default] action create (testbook::default line 3)
[Wed, 09 May 2012 03:29:11 +0000] INFO: ruby_block[loaded_default] called
[Wed, 09 May 2012 03:29:11 +0000] INFO: Processing ruby_block[loaded_restricted] action create (testbook::restricted line 2)
[Wed, 09 May 2012 03:29:11 +0000] INFO: ruby_block[loaded_restricted] called
```

This shows that all recipes not in the allowed recipes list are restricted from loading, but dependencies are allowed.

***
<small>

* [Opsode Chef](http://www.opscode.com/chef/)
* [Runlist Modifiers](http://community.opscode.com/cookbooks/runlist_modifiers)
* [ML Discussion](http://lists.opscode.com/sympa/arc/chef-dev/2012-03/msg00022.html)

</small>
