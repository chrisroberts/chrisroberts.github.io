---
title: "The magic text book"
date: 2015-03-13 06:26:48 -0700
tags: [ruby, grimoire, batali, librarian, depsolver, constraints, dependencies]
---

A new approach at dependency solving.

## Grimoire

A couple years ago I was at the [Chef][6] summit, and had a session with some of my workmates
on cookbook dependency resolution. The result of the session was less than ideal, with
trees being forced as the focus and the sight of the forest lost. It wasn't too surprising.
Resolution was already an issue, and it's hard to sway the conversation from
an issue that was punching people in the face for the one stabbing them in the back.
Compounding this was the fact that I had no solution for the problem, just some ideas
on how it could be fixed.

### Problem

The problem is easily demonstrable via the classic infrastructure repository layout. Reasons
for using the infrastructure repository approach are many, but it's not really relevant
here. What's important is that we aren't solving for a single cookbook. We want to solve
for an entire infrastructure, and this is where existing tools like [Librarian][1] and [Berkshelf][2]
come up short. They provide a single "best" path resolution given a set of dependencies
and constraints. This is fine for a single node. But at the infrastructure level what we
really want is _all_ the cookbooks available within the defined set of dependencies and
their constraints.

Since both [Librarian][1] and [Berkshelf][2] model themselves very closely on [Bundler][4], an analogy
with [Bundler][4] and [RubyGems][5] seems appropriate. Currently solvers are acting like [Bundler][4]:
they solve for a single application. What we want is a solver that solves for [RubyGems][5]:
all the gems that may be required by all the applications. In our case, an application
is equivalent to a node, and [RubyGems][5] is equivalent to our Chef Server.

### Where to start

So, this all came up awhile ago, and has been mostly ignored as a nuisance, but really
annoying when things either can't resolve, or multiple versions may be required.
With [supermarket][7] finally adding support for full manifest, I wanted to start using it within
[Librarian][1]. The project hasn't been updated for awhile, and it looks like the main
contributor ([yfeldblum][8]) is no longer employed with the owning organization. I pinged to
see if maybe I could take over maintanence, start adding support for the new features
available from the [supermarket][7] and start adding some solutions for this infrastructure
resolution problem. When no reply came, I looked at just forking and starting a new
project based on [Librarian][1] since the code is all MIT licensed. But I had also been
reading and browsing around looking at different depsolvers, different implementation
approaches. Why not take a crack at one of these different ideas? I mean, if it totally
crash landed, I could always just fall back to forking [Librarian][1].

### The text book

I had been reading and searching on different solvers around, and how one implementation
differed from another. It would be great to have something I could just build on top
of and not have to create something new. During my searching I stumbled across the
dependency resolver used with [meteor][9]. It uses ideas from the [A* algorithm][10] to generate
resolution paths. And one of the really cool things that it did was only provide updates
to libraries that _required_ updates unless it had been requested. This was something
that always bugged me about all the current tools I have been using ([bundler][4], [librarian][1], etc)
and was added to the list of "things I would like".

Keeping the library as pure [Ruby][11] was important, mainly for portability (more specifically
portability between [Ruby][11] platforms). I read through the [meteor][9] code to get a rough idea
of how they mutated the algorithm for solving use, and read lots more on the [A* algorithm][10]
itself, building slim implementations and playing around with the results. Once it seemed
like a pretty comfortable idea, I took a crack at writing a generic solver library.

### Requirements

#### PriorityQueue

The most important requirement for the solver is a priority queue. There's a few libraries
around that provide priority queues, but I only need a few specific behaviors and I want
something that I can easily optimize. Since it might end up useful elsewhere in the future,
I popped open the [Bogo][12] library and added in a new class: `Bogo::PriorityQueue`. It has
a few features/restrictions:

* Sorts queue based on score
* Sorting can be ascending or descending
* Scores can be provided as a `Numeric` or a `Proc` providing a `Numeric`
* An instance cannot exist within queue multiple times

The `Proc` support for scoring was added just because it could be added. It currently
isn't being used anywhere, and for good reason. With `Proc` based scoring, we have
to resort the queue prior to every `PriorityQueue#pop` since the score value may
have changed. It still seems like it may be useful at some point, for some thing, so
why not throw in support for it while we're there.

#### Requirement of a Dependency for a Version

We also need a way to represent a constraint requirement, a dependency requirement,
and a version. As luck would have it, [Ruby][11] ships with [RubyGems][5] which means we already
have these things available from the [RubyGems][5] library. So lets just hijack these classes
and use them instead of re-inventing them and most likely providing less features.

_NOTE: These still need some work, specifically grimoire providing an interface the class *must*
adhere to so they can easily be swapped._

### Build the stupid thing already!

With the requirements accounted for, lets populate up the library with the things we
need. We'll add a `Unit` class to represent a "thing" that has a "version". A
`RequirementList` that provides names and version constraints. A `System` that holds
all the `Unit`s we have available. A `Solver` to do the solving. And finally,
a `UnitScoreKeeper` that can provide a score for a given `Unit`.

Alright, that was easy! So we end up with a `System` that we populate up with `Unit`s
generated from where ever the actual implemention application decides. That gets passed
to the `Solver` along with the `RequirementList`. The first thing the `Solver` does is
build its "world". The world building is merely trimming down the `Unit`s defined within
the `System` to the set allowed based on the constraints provided within the `RequirementList`
(and the dependency chains).

_Wait: This world thing seems a lot like the "solving for [RubyGems][5]" thing we were talking about earlier..._

Once the world has been defined, it's time to actually solve. By default the solver will
discover valid solution paths and provide a `PriorityQueue` ordered by "best" solution, which
in the default case is the latest possible version available within the applied constraints.
This behavior is on par with the other solvers. Great, but lets do something with a little more
wow.

When we initialize the `Solver`, we can provide a `UnitScoreKeeper`. Within grimoire the
`UnitScoreKeeper` is just an abstract class, relying on the application to provide
the proper scoring behavior. It has a single method, which is provided a `Unit`. The
result is a score for that given `Unit` based on what ever criteria is important to
the application. When the `Solver` generates solution paths, it starts by creating
a `PriorityQueue` for each `Unit` name, and then populates the queue with every
instance with that name available. The `UnitScoreKeeper` allows the application to
suggest to the solver to prefer one version of another based on the score provided.
This allows doing things like weighting versions currently in use, so they are
not updated unless it's required, giving a "least impact" style of updates. Awesome!

### Generic library

The result of this work is a generic library that applications can build off to
create specialized solvers. It allows these specialized solvers to:

* Find single path resolution
* Find single path weighted resolution
* Find all `Unit`s required to solve for a given `RequirementList`

\o/

- https://github.com/spox/grimoire

[1]: https://rubygems.org/gems/librarian "A framework for bundlers"
[2]: https://rubygems.org/gems/berkshelf "Manages a Cookbook's, or an Application's, Cookbook dependencies"
[3]: https://rubygems.org/gems/chef "A systems integration framework"
[4]: https://rubygems.org/gems/bundler "Manages an application's dependencies through its entire life"
[5]: https://rubygems.org "Your community gem host"
[6]: https://chef.io "A systems integration framework"
[7]: https://github.com/chef/supermarket "Chef's community platform"
[8]: https://github.com/yfeldblum "Jay Feldblum"
[9]: https://www.meteor.com/version-solver "Meteor version solver"
[10]: https://en.wikipedia.org/wiki/A*_search_algorithm "A* search algorithm"
[11]: https://ruby-lang.org "Ruby programming language"
[12]: https://rubygems.org/gems/bogo "Helper libraries"