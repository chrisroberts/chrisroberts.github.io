---
title: Building crazy with AttributeStruct
date: 2016-01-27 11:03:38
tags: [ruby,azure,sparkle_formation,attribute_struct]
---

Building support for Azure into SparkleFormation required the
ability to programatically generate complex nested strings. Lets
have a quick look at the crazy needed, and solution discovered.

## Source of the crazy (or why JSON should be data)

AttributeStruct is a pretty fun library. It works great for programmatic generation of complex
data structures and is continually put to the test supporting the SparkleFormation library. One
of the recent additions to the SparkleFormation library was provider specific support for Azure
Resource Manager (ARM) deployment templates. The templates are much the same as you would expect
coming from other Orchestration API templates. Parameters block, outputs block, and of course the
resources. Resources were slightly different, being a different underlying type than other templates
(Array vs. Hash). Things were looking pretty great implementation wise until it came time to tackle
intrinsic functions.

If you're not sure what exactly an intrinsic function is, that's okay. It's a pretty simple concept.
They are functions the template can request during "runtime" when the orchestration API is processing
the template. For example, if you wanted to join a number of strings in a CloudFormation template you
would provide the following data structure:

```json
"my_value": {
  "Fn::Join": [
    ", ",
    [
      "one",
      "two",
      "three"
    ]
  ]
}
```

When CloudFormation processes the template, it will process the `Fn::Join` and pass the values to
a join function and set the resulting value (`"one, two, three"`). HEAT templates use the same
kind of approach, just with different function names. This approach is great because it makes
processing the data, as well as generating the data, very straight forward. Now, we can compare
the same example written for ARM:

```json
"my_value": "[concat('one', ',', 'two', ',', 'three')]"
```

With ARM, instead of a proper data structure defining the desired function to be run with the
list of arguments to be provided, it is all defined within a single string identified by the
starting `[` character and trailing `]` character. At a glance, this looks great. So much
easier to write than all that JSON data structure stuff for the AWS template.

## Your happiness is wrong

If looking at that string defining a function to be evaluated at runtime makes you happy because
it's less complex data structures to construct within your JSON, you need to stop. You should
not be writing JSON. Computers write JSON. It is a _serialization_ format. Not a _user_ format.

It gets worse coming from a generation point of view. In SparkleFormation, helper methods are
provided within the DSL for building the required data structures used within templates. Continuing
with the examples above, the template snippet for a CloudFormation template would look like:

```ruby
my_value join!('one', 'two', 'three', :options => {:delimiter => ', '})
```

When that is compiled, we get the same result as the raw CloudFormation JSON above. It's very straightforward
because we are simply building the underlying data structures. When we look at the ARM example,
things get murky. Extremely murky.

## Breaking down the problem

The implementation of intrinsic functions within ARM create multiple problems within SparkleFormation.
One option is to just be lazy, and let authors write their code strings and not provide helpers. Yet this
approach is orthgonal to the fundamental principle of the SparkleFormation library: DRY and concise
templates. SparkleFormation is _supposed_ to remove the grind from building templates, and not providing
helpers for generating intrinsic functions would be considered a failure in the implementation for the
provider. So, lets tackle the problems:

### Function chaining

ARM intrinsic functions can be chained. For example, to get the region of the current resource
group we would write:

```json
"[resource_group().location]"
```

To write this within SparkleFormation, I expect it to look like:

```ruby
resource_group!.location
```

Using AttributeStruct instances gets us part of the way there. We can use them to build an underlying
Hash data structure allowing only a single key which provides the requested method name. We can chain
method calls easily and when processing simply print the key value of the current data structure, and the
key of it's child data structure, and so on until the leaf has been reached. Easy! This is customized
behavior, so a new class must be introduced: `SparkleFormation::FunctionStruct`.

The new type provides the custom dumping functionality returning a `String` type. Seems great, but we're
not out of the woods yet. Right now, we get an underlying data structure that looks like:

```ruby
{"resource_group" => {"location" => nil}}
```

and with that we can chain our keys together to get the desired string:

```json
"resource_group().location"
```

But we're still missing part of the requirement: Start and end characters flagging this as an intrinsic
function. AttributeStruct provides hierarchy information, which ends up being perfect for this situation.
Since the root `SparkleFormation::FunctionStruct` is the only instance in the chain that needs to customize
it's dump behavior (adding start and end characters for flagging), adding a quick check to the dump for
root status allows wrapping the final string with `[` and `]` to get the desired result:

```json
"[resource_group().location]"
```

### Function parameters

The method chaining problem is solved but a new wrinkle gets added: parameters. Many of the available
intrinsic functions accept or require parameters. Compounding this problem is that some of these
parameters may be other intrinsic functions. Lets look at a simple example:

```json
"[concat('region: ', resource_group().location)]"
```

If we look at what would be expected in a SparkleFormation template it is immediately obvious how things
will break in this use case:

```ruby
concat!('region: ', resource_group!.location)
```

With the implementation thus far, an error would be encountered because the `SparkleFormation::FunctionStruct`
does not properly allow parameters. Customizing the initializer and `method_missing` method allows storage
of parameters provided to function calls which solves this problem easily enough. The `SparkleFormation::FunctionStruct`
holds those parameter values internally, and properly formats and dumps them when requested. Now when we
dump the structure, we get:

```json
"[concat!('region: ', [resource_group().location])]"
```

It's close, but not really what we're after. In fact, it's not at all what we are after because this will just
produce an error. So how can we prevent parameter values from assuming they are root and wrapping themselves? Since
the `SparkleFormation::FunctionStruct` receives the parameter collection and stores it until a dump is requested, lets
just do a quick pre-process on the list. If any `SparkleFormation::FunctionStruct` instances are detected, we can set
the parent link to the current `SparkleFormation::FunctionStruct`. This does two things:

1. Provides expected hierarchy context to the struct
2. Prevents automatic wrapping on dump

And the result:

```json
"[concat('region: ', resource_group().location)]"
```

Perfect! Properly generated intrinsic function String type from programmatic generation with SparkleFormation.

### Stupid return types

Excited to have working intrinsic function generation, I slapped together a template just to try things out in
action. The template itself was extremely simple:

```ruby
SparkleFormation.new(:dummy, :provider => :azure) do
  value add!(1, 2)
end
```

which provided the result:

```json
{
  "value": "[add(1, 2)]"
}
```

Okay, lets get a little more complex and nest some functions:

```ruby
SparkleFormation.new(:dummy, :provider => :azure) do
  value add!(1, add!(3, 4))
end
```

which provided the result:

```json
{
  "value": "[add(1, add(3, 4))]"
}
```

This is all looking really great. Within the intrinsic functions documentation page there are a few
pretty complex examples. Lets take one of those and give it a test. Using this example:

```json
"[reference(concat('Microsoft.Storage/storageAccounts/', parameters('storageAccountName')), providers('Microsoft.Storage', 'storageAccounts').apiVersions[0]).primaryEndpoints.blob]"
```

lets convert this into SparkleFormation:

```ruby
SparkleFormation.new(:dummy, :provider => :azure) do
  value reference!(concat!('Microsoft.Storage/storageAccounts/', parameters!('storageAccountName')), providers!('Microsoft.Storage', 'storageAccounts').apiVersions[0]).primaryEndpoints.blob
end
```

Right away I know this is going to fail. The `SparkleFormation::FunctionStruct` has no support for Array or Hash accessors via `[]`. Adding in some functionality to handle the accessors and
we're on our way. I run the SparkleFormation version of the example and all the excitement around solving this crazy is lost:

```json
{
  "value": "[blob]"
}
```

At this point I assumed I must have broken something. My previous addition test worked brilliantly but now I seem to be losing the entire chain. I double back to the original example
and running it provides the expected result. Dumbfounded, I started reducing my complex example until I reached a point that the generated output matched expections. I reached that point
when the template looked like this:

```ruby
SparkleFormation.new(:dummy, :provider => :azure) do
  value reference!(concat!('Microsoft.Storage/storageAccounts/', parameters!('storageAccountName')), providers!('Microsoft.Storage', 'storageAccounts'))
end
```

It was during this reduction that the obvious answer finally hit me. Once I had removed the end method chaining from the `reference!` call, the `reference` function turned up in the
output, but with one oddity. The second parameter was just `apiVersions`. The underlying problem is the actual return value from _the entire_ right side expression. Duh! What we
need is `value` to be set to the root `SparkleFormation::FunctionStruct` defined within the right hand expression, not the final result which is the _last_ `SparkleFormation::FunctionStruct`.
All we need is to change Ruby's evaluation behavior and return back the value we want. No problem!

#### The dumb solution

The dumb solution is no solution, because it's dumb and the solution doesn't exist. But my first idea was: "lets modify what Ruby returns from evaluation of the right side of the expression"
which led to lots of searching, reading, testing, more searching, more reading, more testing, and on and on until I just gave up. This was looking to be a serious problem that might not
be "easily" solvable without some seriously bad idea hack. Defeated, I moved on to some other tasks and left things for a bit.

#### The less dumb solution

Not thinking about a problem seems to be a great way to find solutions to problems. Doing other mundane things and an idea popped into my head. AttributeStruct is already doing
behavior modification via `method_missing` to provide the data generation DSL behavior. When a value is being set into the underlying data structure of the instance, a quick
check on the type would allow modifying the result if required. Basically, if the value being set is a `SparkleFormation::FunctionStruct`, we can request the root of that structure
and set that, rather than the leaf value that is the actual result. After some modifications to the SparkleFormation::SparkleStruct to perform detections on set, I kick off
the original complex example:

```ruby
SparkleFormation.new(:dummy, :provider => :azure) do
  value reference!(concat!('Microsoft.Storage/storageAccounts/', parameters!('storageAccountName')), providers!('Microsoft.Storage', 'storageAccounts').apiVersions[0]).primaryEndpoints.blob
end
```

and the expected result is generated:

```json
{
  "value": "[reference(concat('Microsoft.Storage/storageAccounts/', parameters('storageAccountName')), providers('Microsoft.Storage', 'storageAccounts').apiVersions[0]).primaryEndpoints.blob]"
}
```

### A recap

1. JSON should be data structures, not code strings
2. We have code strings, so lets figure something out
3. AttributeStruct can be adapted to programmatically generate code strings
4. `SparkleFormation::FunctionStruct` is really pretty great.

And the end result: Ability to properly support ARM templates within SparkleFormation.

* http://www.sparkleformation.io
* https://github.com/chrisroberts/attribute_struct
* https://github.com/sparkleformation/sparkle_formation
