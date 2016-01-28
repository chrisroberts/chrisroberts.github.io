---
title: "Serialized Rainbows"
date: 2015-08-12 20:59:31 -0700
tags: [ruby, attribute_struct, sparkle_formation, cfn, cloudformation]
---

A short story on the origin of the SparkleFormation library.

## SparkleFormation - an origin story

SparkleFormation is a Ruby library that was born out of the insanity induced attempting
to understand how anyone could possibly be managing anything of even basic complexity
in CFN using JSON templates. What I found was lots of copy paste nightmare fuel and
like many before me ran screaming. It is utter madness. And for good reason. JSON is
not an interface. It's a serialization format.


> _SIDEBAR:_ If you are attempting to compose and maintain anything in a serialization format, just stop.
I mean, if you're a masochist then more power to you. Hell, some other twisted souls have
already integrated things like ERB into YAML, so you know you're not alone. But if you are
attempting to do real work then start using tools that generate serialized data. Computers
can do an amazingly good job producing serialized data.

Anyway, I had just written AttributeStruct and its only job is to make Ruby produce
simple data types in a DSL style. So I did the only thing a person staring into the great
abyss could possibly do. I made a tool to generate JSON. Toss in some simple conventions
to allow for reusability, some helper methods to generate common data structures, and
SparkleFormation rose from the black abyss.

## SparkleFormation - the early years

SparkleFormation got legs. It got a knife plugin (knife-cloudformation) providing a CLI
interface to the library itself. Features started getting added. More helper methods.
More reusability features and along with those features new patterns that emerged. New
stack templates could be created with a few lines, and all that previously copy/pasted
JSON was DRY easy to read Ruby.

Then, it happened. The templates were getting so easy to create that stacks previously
deemed too complex were now normal. And limits started to be hit. Resource limits,
parameter limits, output limits. The garden suddenly lost its life and vibrance. Working
around limitations introduced bad patterns to get things moving. I knew of nested
stacks, but they were _hard_. In the context of SparkleFormation, they were really hard.
It introduced new required functionality of remote template storage to even begin to
work, and integrating that into the template generation was hard and took some time
along with a few tries to get right.

### Original Nesters

The original nesting functionality was very simple. It based itself on the `--apply-stack`
option provided via the knife-cloudformation plugin tooling which itself was a manual
implementation of functionality provided by nested stacks. The `--apply-stack` functionality
goes something like this:

> Given an output from a stack (StackA)

```json
...
  "Outputs": {
    "ElbDns": {
      "Description": "Address of ELB",
      "Value": "http://output.example.com"
    }
  }
}
```

> When applied to a new stack (StackB), automatically default matching parameters

```json
...
  "Parameters": {
    "ElbDns": {
      "Type": "String",
      "Default": "http://default.example.com"
    }
  }
}
```

This is pretty useful. Stacks can generate values in their outputs. Those outputs can then
be used to automatically populate parameters on new (or updated) stacks. Less interaction
required by the end user. Pretty great. In the above example, applying StackA to the
creation of StackB will automatically update the default value for `ElbDns` to
`"http://output.example.com"`.

This does introduce update issues. If a stack is updated, and its outputs are modified,
any other stacks that were using those outputs are now using stale values. The responsibility
of updating the _other_ stacks rests on the end user, which introduces a list of new issues
from knowing what stacks require updates to resolving the dependency ordering if it exists.
Using nested stacks removes this issue since we can reference stack outputs directly within
the template, and any changes to those outputs will automatically trigger an update on any
nested stack referencing those outputs.

And this is how the original nesting behavior works. You nest your first stack, and all
parameters defined in that stack are extracted to the top level stack, then referenced
in the nested stack resource allowing the CLI tooling to continue working and all control
for all the stacks being driven from the "root" stack.

> What?

I know, right? Lets use an example. Pretend I have a SparkleFormation template (TemplateA):

```ruby
SparkleFormation.new(:template_a) do
  ...
  parameters.fubar do
    type 'String'
    default 'FOOBAR'
  end
end
```

Now, we're going to nest that template into another template (Root):

```ruby
SparkleFormation.new(:root) do
  nest!(:template_a)
end
```

When we generate Root JSON temlate we end up with one resource (the TemplateA stack resource)
and one parameter:

```json
...
  "Parameters": {
    "Fubar": {
      "Type": "String",
      "Default": "FOOBAR"
    }
  },
  "Resources": {
    "TemplateA": {
      "Type": "AWS::CloudFormation::Stack",
      "Properties": {
        "Parameters": {
          "Fubar": {
            "Ref": "Fubar"
          }
...
```

The `Fubar` parameter was bubbled up from TemplateA to the Root template. This allows the
Root stack to control the contained template (stack updates) so only a single point of
access is required. While cool, this has _nothing_ to do with the `--apply-stack` talk
that led to these examples. Well, it is related because the parameter bubbling is just
the start. Lets expand TemplateA to include an output:

```json
...
  "Outputs": {
    "SuperFubar": {
      "Description": "Fubar with powers",
      "Value": {
        "Ref": "Fubar"
      }
...
```

Now, lets create a new template (TemplateB):

```ruby
SparkleFormation.new(:template_b) do
  ...
  parameters.super_fubar do
    type 'String'
  end
  ...
end
```

And we update our Root stack to nest in TemplateB:

```ruby
SparkleFormation.new(:root) do
  nest!(:template_a)
  nest!(:template_b)
end
```

Generating the Root JSON template looks the same as before, but with
a new stack resource (TemplateB). The `Parameters` block has not changed
(no entry for `SuperFubar`), but we do see `SuperFubar` in the `Parameters`
section of the stack resource:

```json
...
    "TemplateB": {
      "Type": "AWS::CloudFormation::Stack",
      "Properties": {
        "Parameters": {
          "SuperFubar": {
            "Fn:GetAtt": [
              "TemplateA",
              "Outputs.SuperFubar"
            ]
          }
...
```

What is happening is that SparkleFormation automatically applies the same
behavior `--apply-stack` provides, but does it automatically within the
template. This means that when the `SuperFubar` output values changes in
TemplateA due to a stack update, TemplateB will automatically get triggered
for an update. Everything is wonderful!

### The other stuff

More things came out of this library origin. The Miasma library grew out of
frustrations with the inconsistencies in fog and hijacked work. The sfn
tool grew out of the knife-cloudformation plugin to free the CLI tool of
its knife dependency. And all the functionality bottled up in the sfn
CLI can trace their roots back to a moment of utter JSON induced horror.