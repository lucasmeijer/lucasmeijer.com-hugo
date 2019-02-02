---  
title: "There's no such thing as an API break"  
date: 2019-02-02T01:33:37+01:00
comments: false
---

Semantic Versioning is pretty popular these days. It basically is a convention that gives some meaning
to version numbers. Roughly speaking: if your change breaks API, you should bump the major version,
if you just make an improvement or bugfix bump the minor version.

There's nothing wrong with trying trying to convey a convention that lets authors communicate which
changes are breaking and which not. What can drive me a bit crazy is when this is mistaken for a rock
solid way to know if a change is breaking or not.


### Let's talk about what really is an API break anyway.

In a statically typed language as C#, most people would answer something like:

"when you change/remove the signature of any public method/type"

If you change a bool to an enum, rename a method, change the returntype, all these things would qualify as a breaking change. It all sounds very much like a binary thing. Either something is an API break, or it is not.

I think this is a very romantic way to look at things. (I stole the romantic label from [this great post](https://gist.github.com/jashkenas/cbd2b088e20279ae2c8e))

When you want to predict if a code change will break any of its users, there is nothing binary about that.
Let's take a look at a few real world examples that we deal with regularly at Unity:

- I have a change that improves performance by 10x for most usecases, but there's a niche case that gets 2x slower. Is that an API break?

- I remove a private method that I know is actually used through reflection by a very popular 3rd party package
that thousands of games are using. Is that an API break?

- I add an overload of a method. (or a new type) it could be there's consuming code which now becomes ambiguous
where it wasn't previously.  (Unity's packages ship as source, so we care about both source and binary compatibility) Is that an API break?

- I rename a method, but our script updater is automatically able to "fix up" all consuming code transparently. Is that an API break?

- I have a package that has code & 3d models for driving 2nd world war tanks. Some professor sends me an email
that informs me I got the measurements on the width of one of the tanks a bit wrong, should be 10cm wider. Is that an API break?   (maybe the tank is used by a game, and now the tank can no longer fit through a certain alley)

To me these examples show that looking at this from this black and white perspective is pointless. Instead what
we should do when we are considering making code changes is:

- how many users will this break
- how bad is the break.  are we talking a few % perf in some function, or are we talking game doesn't start.
- how bad is the fix. does the user just have to rename a method, or do they have to rewrite half their game?
- how many users will benefit, and how much

Looking at all these factors you have to make a judgement call. A judgement call on if you want to do the change
at all. If you want to do the change in this release and major bump. or maybe just minor bump.  Or maybe you want
to postpone the change, so you can batch it with a bunch of other changes that also require some user work when she updates.

When you make the change, the only thing you can (and should) do is signal your intent of where this change is on the
grayscale spectrum of breakage. In most cases you have no idea if you actually broke user code, because you don't have
your users code.



### Package Managers

When you want to update a dependency of your project, your package manager (in any language/ecosystem) has to resolve
all dependencies. Maybe your dependency actually depends on a newer version of another dependency. As it tries to figure out
which exact version of all these dependencies to grab, it has to make a reasonable guess on which exact versions will work together.
Having SemVer here is better than nothing. At least there's some intent by the author.   If that set of packages actually works together remains
to be seen though.

Can we do any better?

Unity authors most of the low level packages that our users, and other Unity authored packages consume. It would help us as authors of these low level packages greatly if we'd know if a change we want to make, or made, actually breaks consuming code. 

We're building a CI system that can tell us that. Example:

A very low level package we're working on is called Unity.Entities.  It is consumed by most other new
packages, including say Unity.Rendering.  If we change Unity.Entities, instead of doing a naive scan of the change and say "no this is
not a breaking change as no signatures change",  we actually go and run all packages that we know of that depend on Unity.Entities,
and run _their_ test suite.

The result of one such "compatibility check" could be that some unit test failed. If we intended our change to not be breaking, that is good to know, and we should probably adjust our change to indeed be not breaking.  If we intended it to be a breaking change, then we can automatically inform the authors of Unity.Rendering to let them know there's a new version of Unity.Entities in the works, and that at this moment Unity.Rendering is not compatible with it.

This system would not only be used by us internally, but would also allow us to test our changes against all asset store packages (once we have migrated asset store packages to our new package manager, which will take some time(tm) ), and allows us to notify asset store package authors early when we have some changes in the works that will require some modifications on their end.

All of this puts enormous pressure on each package having the ability to let itself be verified if it still works in an automated way.
I guess that's code speak for: it needs a decent test suite. Making sure all packages have good test suites is hard, but we need to do it anyway,
how else will we know if anything still works?

Maybe instead of using this compatibility information only to inform authors of both packages, we could also use it to have the package manager
make better decisions about which packages are _actually_ compatible with each other, instead of romantic faith in a version number.

### Other ecosystems doing something similar?
I'm not aware of any other package ecosystem having automated compatibility testing working, especially not on the code as it's being worked on in a git repo, and not yet in a release. If anybody knows of an ecosystem having a similar setup, I'd love to hear about it to try to not make the same mistakes (but new ones! ;p)