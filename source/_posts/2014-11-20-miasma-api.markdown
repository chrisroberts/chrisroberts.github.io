---
title: "Miasma - Cloud Models"
date: 2014-11-23 07:46:33 -0700
tags: [ruby, cloud, api, models, fog]
---

New Ruby cloud API library with a specific purpose.

## Cloud Infrastructure API

I've been using [fog][1] for a long time, and have
added a few different features to the library including:

* [Rackspace networks](https://github.com/fog/fog/pull/1761)
* [S3 bucket tagging](https://github.com/fog/fog/pull/2322)
* [DNSimple API token auth](https://github.com/fog/fog/pull/2739)

One of the great things about [fog][1] is that it support _everything_
cloud which means it's super handy for interacting with various
APIs. Especially where certain cloud features may be available
via the API but not an existing UI, it's very simple to jump
into an IRB session and start running.

Cloud API coverage is a very important feature within the
[fog][1] library. Yet, some times coverage of all features available
within a specific API is not important. At times the power that
comes from consistency can be greater than that of coverage.

## Cloud Models

[Fog][1] includes the concept of infrastructure resource models. The
concept is pretty simple. Define a model for a given resource,
then wire in the provider. I've used the `File` and `Server`
models with relative success, and the concept is sound. Having
built out the [knife-cloudformation][2]
plugin targeted at AWS, common models really make sense to
prevent lots of provider specific code branches.

### Orchestration model

With the addition of the [Heat][4] service within [OpenStack][5]
the need for generic models was highlighted. If a generic model
had been used for [knife-cloudformation][2], the addition of [Heat][4]
support would have been as simple as the provider compatible implementation.
[Fog][1] had an `Orchestration` model some what defined, but it was
only within `OpenStack`.

So I built out the abstract model and worked AWS CFN and RS orchestration
support into them:

* [Add abstract orchestration modeling](https://github.com/fog/fog-core/pull/45)
* [Implement orchestration models](https://github.com/fog/fog/pull/2971)

Once these models were in place, [knife-cloudformation][2] was updated to
use the generic models, and multi-provider orchestration support was achieved.
It cost some features previously available within the knife plugin, but
they were mainly informational features that could be focused on later. The
important milestone was actually using the same code base for interacting
with multiple providers.

Now we have working multi-provider knife orchestration plugin, but it's not
really usable. It's _usable_, but not easily. It requires a custom Gemfile
with multiple custom github references. Internally, and on a small scale,
that's kinda managable. Yet, on any larger scale that's a non-starter. The
changesets within the pull requests are large, and uses a slightly different
modeling approach than in the past (making use of `fog-core` for generic
model definition). And there's the matter of a lack of tests, mainly due
to discussions of migrating test suites but no concrete examples of abstract
tests (defined within `fog-core`) and their corresponding concrete
implementations. Trying to make PoC abstract tests, keeping the PRs up-to-date
and still pushing forward the libraries using these new models ended
up being too much.

### I'll make my own cloud lib, with blackjack...

The knife plugin has/had a subcommand for stack inspection. Ideally, it's to
be used for getting real information from the stack resources information. In
the AWS specific version I had implemented custom flags for doing direct
actions (like getting node IP addresses from auto scale groups). Moving to
generic models meant that I should be able to do this in much the same
way as the orchestration work was done. Use generic models and get provider
specific support under the hood. But the models don't all follow the same
API. Some models provide extra methods that other models don't implement
at all, or are structured in incompatible ways making them provider specific
models. It was here that it felt like the rabbit hole I was running down
to get the behavior I was after was just growing too big to keep a handle
anymore.

I found myself thinking "I should build my own library". And then, "this
is a horrible idea". Yet, some times starting from scratch allows certain
freedoms that aren't available when you have to worry about breaking changes
and keeping things backwards compatible. It also allows for changing core
implementations and I have ideas about some great changes. Still though,
"this is a horrible idea". Some times horrible ideas can have great results.

[Miasma][6] started like many of my PoC libraries:

* Annoyance of current state
* Weeks/months of mulling over different approaches
* Architecture ideas forming and getting thrown away

With all this culminating into a few evenings after work with a cold beer
laying out the "horrible idea". And after the skeleton is defined and a
handful of helpers are implemented it starts to move from "horrible idea"
to just "bad idea". With this fresh core implementation we get some pretty
great things:

* Model down approach
* Light weight library
* Simple access points for provider specific implementations

Instead of attempting to support the cloud API of the provider, we start
at the model, and make the provider support the model. This simple reversal
ensures we keep a consistent model regardless of the underlying provider.

We also get to re-pick our underlying libraries, which is an important
part of the new implementation. [Nokogiri][7] which is an awesome library
provides a lot of pain when it's a hard requirement. The need for build
tools, and the time required for compilations start to compound as it
becomes a requirement within deployed tooling. So, lets make it an actual
choice! With `multi_json` allowing for lazy requirements on the JSON library,
we'll use `multi_xml` to do the same for the XML library.

Slowly moving from "horrible idea" to "bad idea", all these internal implementation
changes keep nudging the scale. Yet, all these things are just abstracts. At this
points we could be hovering around "neutral idea". Everything sounds awesome in
theory. Concrete implemenations actually make it awesome. So lets make some awesome.

### Miasma Compute

So I started with an AWS compute provider. Low hanging fruit for the PoC. Well, not
really. AWS signature v4 is kind of a pain. And that took some cycles getting everything
formatted correctly, and the signature correct. Good for "experience" but not super
helpful if this doesn't pan out. Oh well, onward and upward!

With the signature4 implementation done, it took about 30 minutes to get the
API parts implemented for the basic compute model. Great! Now we need another provider
to actually validate this modeling approach is valid. So lets lay down a Rackspace
provider. Cycle to get RS authentication in working order, and spend an hour
getting the implementation done, and reworking some things that weren't really
generic.

End result: concrete awesome!

### Not a horrible idea (reclassification)

So now the benefits of starting from scratch are showing. It's proving that we
get some great things. So lets power forward! And now we have a new model
based cloud API library that's starting to come together.

## In Action

```ruby
require 'miasma'

compute = Miasm.api(
  :provider => :aws,
  :type => :compute,
  :credentials => {
    ...
  }
)
compute.servers.all.each do |srv|
  puts "SRV #{srv.id} -> #{srv.address}"
end
```

and we can swap out to OpenStack, and all that changes is the compute
configuration:

```ruby
require 'miasma'

compute = Miasma.api(
  :provider => :open_stack,
  :type => :compute,
  :credentials => {
    ...
  }
)
compute.servers.all.each do |srv|
  puts "SRV #{srv.id} -> #{srv.address}"
end
```

The library itself is still in a beta state. Some models are still yet to be
defined, but they are quickly getting implemented. The [knife-cloudformation][2]
plugin has been migrated fully over to this new library, which had the added
feature of being able to cut a proper release as it removed the custom Gemfile
requirement. \o/

[1]: http://fog.io "The Ruby cloud services library"
[2]: https://github.com/hw-labs/knife-cloudformation "Knife plugin for cloud formation"
[3]: https://github.com/sparkleformation/sparkle_formation "Ruby orchestration template toolkit"
[4]: https://wiki.openstack.org/wiki/Heat "OpenStack Orchestration"
[5]: http://www.openstack.org "Open Source Cloud Computing Software"
[6]: https://github.com/chrisroberts/miasma "Cloud modeling library"
[7]: http://www.nokogiri.org "XML is like violence - if it doesn’t solve your problems, you are not using enough of it."