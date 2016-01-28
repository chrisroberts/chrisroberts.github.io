---
title: "AttributeStruct - Turtles all the way down"
date: 2014-11-20 07:46:33 -0700
tags: [ruby, hashes, attribute_struct, dsl]
---

Building attribute structures with Ruby.

## The origin story

I'm a big fan of JSON serialization and Hash/Array collection
types in general. Awhile back, I was writing an LWRP for the
lxc cookbook and decided I wanted to do "sub-resources". Basically
nest resources within a "parent" resource. Yet, these nested
resources I also wanted to be properly standalone. To facilitate
this, I simply defined a method instead of an attribute within
my resource file, and cached the name and proc into an instance
variable:

```ruby
# resources/thing.rb
attr_reader :subresources
def initialize(*args)
  @subresources = []
  super
end

...

def other_resource(o_name, o_block)
  o_resource = Chef::Resource::OtherResource.new("custom_other[#{self.name}-#{o_name}", nil)
  o_resource.action :nothing
  @subresources << [o_resource, o_block]
end
```

Now with this `subresources` reader, in the provider I can cycle
the defined resources and provide them within my parent resource:

```ruby
# providers/thing.rb
def load_current_resource
  new_resources.subresources.map! do |subresource, block|
    subresource.run_context = run_context
    subresource.instance_eval(&block)
    subresource
  end
end

action :create do
  ...
  ruby_block "Thing #{new_resource.name} - Run subresources" do
    block do
      new_resource.subresources.each do |subresource|
        subresource.run_action(:create)
      end
    end
    not_if{ new_resource.subresources.empty? }
  end
end
```
Cool, this provides an easy way including the definition of a standalone
resource within a resource (though the real implementation is a little
different than this example). But this sub-resource stuff isn't really
the point here. It got me thinking about this "nesting", and if I wanted
to nest `n` number of resources, with `y` depth of resources nested, what
would that look like? Could it be done in a generic way? This is what led
to the creation of the `attribute_struct` library.

### Let make some magic

At this point, I was still messing around in a resource file using `#method_missing`
and making a complete mess. But, in the mess was something that was starting
to work. The `#method_missing` could provide some of what I wanted, but it was
flat, which meant only a single subresource (no infinite nesting). Time to
break out to a clean slate.

The `attribute_struct` library is mostly just a single file. In the beginning it
was very very small, consisting of just the `#method_missing` definition. The
approach was pretty simple. Underneath the hood, it's all just Hashes. Everything
else is just sugar to make building them, and working with them easier. The
behavior that was defined within an `AttributeStruct` instance is as follows:

* A method call with no argument: request for the value
* A method call with an argument: request to set the value
* A method call with block: set new `AttributeStruct` and execute block within

Pretty simple behavior that results in some pretty great things. This behavior
allows for building deeply nested hashes very easily. Lets look at a simple
example of building a `Hash` directly, and build it using `AttributeStruct`:

```ruby
# direct hash
hash = {
  :a => {
    :b => {
      :c => {
        :d => 'some_value'
      }
    }
  },
  :x => {
    :y => {
      :z => 'other_value'
    }
  }
}
```

```ruby
# via attribute_struct
hash = AttributeStruct.new{
  a.b.c.d 'some_value'
  x.y.z 'other_value'
}._dump
```

Under the hood `AttributeStruct` is using the `#method_missing` magic
to see that `a` is not defined within it's underlying data hash, so it
sets an entry into the data hash with the value being a new instance
of `AttributeStruct` (which when empty `.nil? == true`). This allows
us to automatically create the required nested "hashes" as we traverse
to the final set.

Cool!

### Re-entry

What we end up with is a large (perhaps nested) hash getting created. There
is no "compile" or anything like that happening behind the scenes. The underlying
data hashes are being built in real time, which means that you can easily update
existing state:

```ruby
AttributeStruct.new do
  my.configuration.value 22
  my.configuration do
    other_value 23
  end
end
```

Which results in:

```
{'my' => {'configuration' => {'value' => 22, 'other_value' => 23}}}
```

We can also grab the parent of the depth we are currently at:

```ruby
AttributeStruct.new do
  my.configuration.value 22
  my.configuration do
    other_value _parent.value
  end
end
```

Which results in:

```
{'my' => {'configuration' => {'value' => 22, 'other_value' => 22}}}
```

### Results

The result of the `attribute_struct` library is a slim DSL for building
hashes. It includes other features, like deep merging two `AttributeStruct`
objects, automatically camel or snake casing keys, and folding values
to produce an array when the same key is set multiple times. All of these
things provide a very powerful toolkit for building complex hashes.

Why do we care about Hashes, or even building complex Hashes? Because in the
end, everything is a Hash.

* https://github.com/chrisroberts/attribute_struct
