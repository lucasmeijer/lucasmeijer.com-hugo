---
title: "Whats next after Burst"
date: 2019-01-04T03:34:28+01:00
draft: false
---

In my [previous post](../cpp_unity) I talked about a few fundementals of Unity's Data Oriented Tech Stack (DOTS) future:

- HPC# and Burst compiler
- Job system

At this point in our tech stack, we can generate some really fast code, we can run jobs safely and efficiently. 

I like to refer to this level of our stack as the "game engine engine". Anyone can use this stack to write a game engine. We can. We will. You can too. Don't like ours? Write your own, or modify ours to your liking.

The next layer we're building ontop is a new component system.

### Unity's Component System

Unity has always been centered around the concepts of components. You add a Rigidbody component to a GameObject, it will start falling. You add a Light component to a gameobject, it will start emitting light. Add an AudioEmitter component, and the gameobject will start producing sound.

It's a very natural concept for programmers and non programmers alike, and easy to build intuitive UI's for.  I'm actually quite amazed how well this concept has aged. It has aged so well, that we want to keep it.

What has not aged well is how we implmented our component system. It was written with an object oriented mindset. Components and GameObjects are "heavy c++" objects. Creating/destroying them requires a mutex lock to modify the global list of id->objectpointers. All gameobjects have a name. Each one gets a C# wrapper object that points to the C++ one. That C# object could be anywhere in memory. The C++ object can also be anywhere in memory. Cache miss galore. We try to mitigate the symptoms as good as we can, but at the end of the day it's still lipstick on a pig.

What we've done is build a new component system, that from a user point of view has the same properties (add a rigidbody component, and the thing will fall), but is implemented with a data oriented mindset.

This new component system we call Entity Component System (ECS). I'm not a big fan of the name, but too late to go back now :). Very roughly speaking, what was GameObject is Entity in the new system. Good riddance, we never liked the name GameObject anyway. Components are still called components. So what's different? The data layout.

### Let's look at some common data access patterns

A user written component in traditional Unity might look like this:

{{< gist lucasmeijer 9fbc348c113c81449ce39f858f2b78a3 >}}

This pattern comes back over and over. A user written component has to find one or more other components on the same gameobject, and read/write some values on it.

There's a lot of things wrong with this.

- the Update() method gets called for a single orbit component.  the next Update() call might be for a completely different component, likely causing this code to be evicted from the cache the next time it has to run this frame for another Orbit component

- Update() has to use GetComponent<RigidBody>() to go and find its rigidbody. (It could be cached instead, but then you have to be careful about the rigidbody component not being destroyed).

- The other components we're operating on are in a completely different places in memory

The datalayout ECS uses recognizes this is a super common pattern, and optimizes memory layout to make operations like this be fast.

### ECS Data Layout

ECS groups entities by "the set of components they have". It calls such a set an archetype. An example of an archetype is:  "Position & Velocity & Rigidbody & Collider". ECS allocates memory in chunks of 16k. Each chunk will only contain entities of a single archetype.

Instead of having the user component searching for other components to operate on at runtime, per orbit instance, in ECS land you have to statically declare "I want to run some operations on all entities that have both a Velocity and a Rigidbody and an Orbit component. To find all those entities, we simply find all archetypes that match that "component search query". 
Each archetype has a list of Chunks where entities of that archetype are stored. We loop over all those chunks, and inside each chunks, we're doing a linear loop of tightly packed memory, to read and write the component data.
This linear loop that runs the same code on each entity also makes for a likely vectorization opportunity for burst.

In many cases, this process can be trivially split up into several jobs, making the code operating the ECS component run on nearly 100% core utilization.

ECS does all this work for you, you just need to supply the code that you want to run on each entity.
(You can do the chunk iteration manually if you want to though.)

When you add/remove a component from an Entity, it switches archetype. We move it from its current chunk to a chunk of the new archetype, and backswap the last entity of the previous chunk to "fill the hole".

ECS components can also statically declare what they intend to do with the component data. ReadOnly or ReadWrite. By promising (the promise is verified) to only read from the Position component, ECS can get more efficient scheduling of its jobs. Other jobs that also want to read from the Position component will not have to wait.

This data layout also allows us to deal with a long standing frustration we've had, which are load times and serialization
performance. "loading/streaming" ECS data say for a big scene is not much more than just loading raw bytes from disk and using them as is.

This is the reason the [mega city demo](https://www.youtube.com/watch?v=j4rWfPyf-hk): loads in a few seconds on a phone.

Sidenote: in pre-ECS, we have all the data nicely tightly packed on disk, but then we start allocating
objects all over the place, and copy the data into all those locations. The irony hurts.

### HPC#, Burst, ECS. Awesome, but where's my game engine?

The next layer we need to build is very big. it's the "game engine" layer composed of features like "renderer", "physics", "networking", "input", "animation", etc. This is roughly where we are today. We have started work on these pieces, but they won't be ready over night.

That might sound like a bummer. In a way it is, but in another way it is not. Because ECS and everything built ontop of it
is written in C#, it can run inside of traditional Unity. (In fact, today that's the only environment it runs in, but that will change). By virtue of running inside of traditional Unity, you can write ECS components that use pre-ECS functionality. There is no pure ECS mesh drawing system right now. However we can write an ECS MeshRenderSystem that uses the pre-ECS Graphics.DrawMeshIndirect API as an implementation, while we wait for a pure ECS version to show up. (This is exactly the technique that MegaCity demo uses. Loading/Streaming/Culling/LODding/Animation is done with pure ECS systems, but the final drawing is not).

So you can mix & match. What's great about that is you can already reap the benefits of burst codegen, and ECS performance
for your game code, instead of having to wait for Unity to ship pure ECS versions of all subsystems. What's not great about it is that in this transition phase, you can see and feel this friction that you're "using two different worlds that are glued together".

We will ship all our ECS HPC# subsystems as source in packages. You can inspect, debug, modify each subsystem, as well as have more fine grained control over when you want to upgrade which subsystem. (you could upgrade physics subsystem package without upgrading anything else).

That's it for now!

There are no comments on this website, but please feel welcome to join the discussion on twitter [@lucasmeijer](https://www.twitter.com/lucasmeijer)