---
title: Tracing SparkleFormation Templates
date: 2019-01-27 10:07:23
tags: [ruby,sparkle_formation,sfn,tracing]
---

SparkleFormation templates are generally composed of items from multiple
locations: builtins, local SparklePack, remote SparklePacks, etc. Complex
templates can be difficult to debug when a user may not have a comprehensive
understanding of all the underlying SparklePacks.

## Template Audit Logs

To help provide template composition information the first thing needed is
a way to record all the items used to build the template. An AuditLog has
been introduced into the SparkleFormation library and a new Record is added
to the AuditLog for every item used to compose the template. With this
AuditLog in place, it's easy to unpack and display this information to the
user visualizing how the template has been composed.

## Tracing and sfn

With an AuditLog implementation in place within the SparkleFormation library,
a new `trace` command has been added to the sfn CLI which provides a user
interface for displaying the AuditLog Records. So lets start with a simple
example utilizing only a single builtin dynamic:

~~~ruby
SparkleFormation.new(:dynamics_only) do
  dynamic!(:ec2_instance, :test)
end
~~~

Using the `trace` command from the sfn CLI the underlying composition is provided:

~~~shell
$ sfn trace --file dynamics_only
[Sfn]: Trace information:
[Sfn]: -> Template - dynamics_only
[Sfn]:  |  source: /home/spox/workspace/sparkle/sparkleformation/dynamics-only.rb
[Sfn]:  |  caller: template
[Sfn]:  |-> Dynamic - ec2_instance
[Sfn]:  | |  source: builtin
[Sfn]:  | |  caller: /home/spox/workspace/sparkle/sparkleformation/dynamics-only.rb @ 2
~~~

In this simple example information about the source template being processed
is provided first, followed by the builtin dynamic used and the location in
the template which invoked the dynamic (line 2). As the output shows even
in this simple example the composition output also includes indentation which
provides extra context on where the caller is located.

### Multi-file template

In this example a registry and component item is defined and the template
is composed of the two items. First, the registry item:

~~~ruby
SfnRegistry.register(:key_name) do
  "MyDefaultKey"
end
~~~

Next, the component item:

~~~ruby
SparkleFormation.component(:base) do
  parameters.stack_creator.type "String"
  dynamic!(:ec2_instance, :test).properties do
    key_name registry!(:key_name)
  end
  outputs.stack_creator.value ref!(:stack_creator)
end
~~~

Finally, the template:

~~~ruby
SparkleFormation.new(:multi_item).load(:base) do
  dynamic!(:ec2_volume, :test)
end
~~~

When the trace is run against this template not only are the different
items used to compose the template shown, the nested items are also displayed:

~~~shell
[Sfn]: Trace information:
[Sfn]: -> Template - multi_item
[Sfn]:  |  source: /home/spox/workspace/sparkle/sparkleformation/multi_item.rb
[Sfn]:  |  caller: template
[Sfn]:  |-> Component - base
[Sfn]:  | |  source: /home/spox/workspace/sparkle/sparkleformation/components/base.rb
[Sfn]:  | |  caller: template
[Sfn]:  | |-> Dynamic - ec2_instance
[Sfn]:  | | |  source: builtin
[Sfn]:  | | |  caller: /home/spox/workspace/sparkle/sparkleformation/components/base.rb @ 3
[Sfn]:  | |-> Registry - key_name
[Sfn]:  | | |  source: /home/spox/workspace/sparkle/sparkleformation/registry/key_name.rb @ 1
[Sfn]:  | | |  caller: /home/spox/workspace/sparkle/sparkleformation/components/base.rb @ 4
[Sfn]:  |-> Dynamic - ec2_volume
[Sfn]:  | |  source: builtin
[Sfn]:  | |  caller: /home/spox/workspace/sparkle/sparkleformation/multi_item.rb @ 2
~~~

### Nested template

The `trace` command will also properly handle template nesting. Using the
template from the previous example, a simple template is used for nesting:

~~~ruby
SparkleFormation.new(:nested) do
  dynamic!(:ec2_instance, :test)
  nest!(:multi_item)
end
~~~

Running the trace on the new template displays the builtin dynamic as well as
the nested template with all items used to generate the nested template. The
indentation provides context on where the source files for each item is located
and all the items they provide.

~~~shell
[Sfn]: Trace information:
[Sfn]: -> Template - nested
[Sfn]:  |  source: /home/spox/workspace/sparkle/sparkleformation/nested.rb
[Sfn]:  |  caller: template
[Sfn]:  |-> Dynamic - ec2_instance
[Sfn]:  | |  source: builtin
[Sfn]:  | |  caller: /home/spox/workspace/sparkle/sparkleformation/nested.rb @ 2
[Sfn]:  |-> Template - multi_item
[Sfn]:  | |  source: /home/spox/workspace/sparkle/sparkleformation/multi_item.rb
[Sfn]:  | |  caller: /home/spox/workspace/sparkle/sparkleformation/nested.rb @ 3
[Sfn]:  | |-> Component - base
[Sfn]:  | | |  source: /home/spox/workspace/sparkle/sparkleformation/components/base.rb
[Sfn]:  | | |  caller: template
[Sfn]:  | | |-> Dynamic - ec2_instance
[Sfn]:  | | | |  source: builtin
[Sfn]:  | | | |  caller: /home/spox/workspace/sparkle/sparkleformation/components/base.rb @ 3
[Sfn]:  | | |-> Registry - key_name
[Sfn]:  | | | |  source: /home/spox/workspace/sparkle/sparkleformation/registry/key_name.rb @ 1
[Sfn]:  | | | |  caller: /home/spox/workspace/sparkle/sparkleformation/components/base.rb @ 4
[Sfn]:  | |-> Dynamic - ec2_volume
[Sfn]:  | | |  source: builtin
[Sfn]:  | | |  caller: /home/spox/workspace/sparkle/sparkleformation/multi_item.rb @ 2
~~~

The new trace functionality is available in sfn starting with the 3.1.2
release.
