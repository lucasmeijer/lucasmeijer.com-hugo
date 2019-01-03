---  
title: "C++, C# and Unity"  
date: 2019-01-03T01:33:37+01:00
comments: false
draft: true
---

A lot has been [said](figure out this footnote link) and [written] (https://aras-p.info/blog/2018/12/28/Modern-C-Lamentations/)  lately about the game industry's "C++ is not changing into the thing we need" feelings. Valid criticism on various things that make C++ not a great language for games (or at all), and [valid counter criticism](https://medium.com/@pat_wilson/get-your-shit-together-6ccbfd6bb755) of "well why dont you guys participate in the open-for-everyone process of designing C++ instead of bitching from the sidelines".

Let's talk about the place C++ will have at Unity:

Unity and advanced game programmers' problem at the end of the day is that they need to provide an executable with instructions the target processor can understand, that when executed will run the game.

For the performance cricital part of our code, we know what we want the final instructions to be. We just want an easy way to describe our logic in a reasonable way, and then trust and verify that the generated instructions are the ones we want.

C++ is not great at this task. I want my loop to be vectorized, but a million things can happen that might make the compiler not vectorize it. It might be vectorized today, but not tomorrow if a new seemingly innocent change happens.
Just convincing all my C/C++ compilers to vectorize my code at all is hard.

We took a step back and figured: wow it's kind of crazy the only popular way to generate machine instructions somewhat comfortably is C/C++. (There's Go and Rust these days too, which is great, this space can use some competition). 

We decided to make our own "reasonably comfortable way to generate machine code", that checks all the boxes that we care about. We could spend a lot of energy trying to bend the C++ design train a little bit more in a direction it would work a little bit better for us, but we'd much rather spend that energy on a toolchain where we can do all of the design, and that we design exactly for the problem that game developers have.

What checkboxes do we care about?

- **Performance is correctness**. I should be able to say "if this loop for some reason doesn't vectorize, that should be a compiler error, not a 'oh code is now just 8x slower but it still produces correct values, no biggy!'"

- Cross-architecture. The input code I write should not have to be different for when I target iOS than when I target Xbox. (this sounds like a no brainer, but after pulling your hair out getting a C++ compiler to reliably generate the instructions you want, it's very common to just tableflip, and write the instructions you want in assembler and be done with it)

- We should have a nice iteration loop where I can easily see the machine code that is generated for all architectures as I change my code. The machine code "viewer" should do a good job at teaching/explaining what all these machine instructions do.

- Safety. most game developers don't have safety very high on their priority list, but we think that the fact that it's really hard to have memory corruption in Unity has been one of its killer features. There should be a mode in which we can run this code that will give us a clear error with a great error message if I read/write out of bounds or dereference null

Ok, so now we know what things we care about, next step we need to decide on what the input language for this machinecode generator is. What options do we have:

- Custom language
- Some adaption/subset of C or C++
- Subset of C#

Say what C#? For our most performance critical inner loops?

Yes. C# is a very natural choice that comes with a lot of nice benefits for Unity:

- It's the language our users already use today
- Has [great](https://www.jetbrains.com/rider) IDE [tooling] (https://visualstudio.microsoft.com/) (both editing/refactoring as well as debugging) that C++ programmers often have no idea even was possible. (should do a follow up post on this one day)
- A C#->intermediate IL compiler already exists (the [Roslyn](https://github.com/dotnet/roslyn) C# compiler from microsoft), and we can just use it instead of having to write our own.
- We have a lot of experience modifying intermediate-IL, so it's easy to do codegen and postprocessing on the actual program

I quite enjoy writing code in C# myself. However, traditional C# as a whole is not an amazing language when it comes to performance. While the C# language team, standard library team, and runtime team have been making great progress in the last two years, we're still looking at a language where you have no control over where/how your data is laid out in memory, while that is exactly what we need.

On top of that the standard library is oriented around "objects on the heap", and "objects having pointer references to other objects".

That said, if we give up on the most of the standard library, (bye Linq, StringFormatter, List<T>, Dictionary), disallow allocations (=no classes, only structs), no garbage collector, dissalow virtual calls and non-constrained interface invocations, and add a few new containers that you are allowed to use (NativeArray<T> and friends) the remaining pieces of the C# language are looking really good. Remember this is only for your performance critical code. Here's an example from our [mega city demo](https://www.youtube.com/watch?v=j4rWfPyf-hk):

<script src="https://gist.github.com/lucasmeijer/bb5ba6a73340566e9b7273a541d191de.js"></script>

This subset lets us comfortably do everything we need in our hot loops. Because it's a valid subset of C#, we can also run it as regular C# getting errors on out of bounds access, great error messages, debugger support and compilation speeds you forgot were possible when working in C++. We often refer to this subset as HighPerformanceC#, HPC#

## Where are we today?

We've built this code generator / compiler, and it's called Burst. It ships with Unity2018.3. We have a lot of work ahead, but we're already happy with it today. Performance is often better than C++, but comparing performance is not exactly the right thing to do. What matters is what you had to do to get that performance. Example: we took the c++ culling code of our current c++ renderer and ported it to Burst. Performance was the same, but the C++ version had to do incredible gymnastics to convince our c++ compilers to actually vectorize. The Burst version was about 4x smaller, running slightly faster.

To be honest, the whole "you should move your most performance critical code to C#" story also didn't result in everybody internally at Unity immediately buying it. For most of us it feels like "you're closer to the metal" when you use C++. But it's not true anymore. When we use C# we have complete control over the entire process from source compilation down to machine code generation, and if there's something we don't like, we just go in and fix it. No comittee to convince of the value of our usecase, or other concerns to be balanced against.

We will slowly but surely port every piece of performance critical code that we have in C++ to HPC#. It's easier to get the performance we want, harder to write bugs, and easier to work with.

Unity has a lot of different users. Some can enumerate the entire arm64 instruction set from memory, others are happy to create things before getting a PhD in computer science.

Both examples of users benefit as the parts of their frametime that is spent running engine code (usually 90%+) get faster. The parts that are running asset store package runtime code gets faster as vendors adopt HPC#.

Advanced users will benefit ontop of that by being able to also write their own high performance code in HPC#.

Here's a screenshot of Burst Inspector, allowing you to easily see what assembly instructions were generated for your different burst hotloops:

[<img src="../../images/burst-inspector.png"/>](../../images/burst-inspector.png)

## Optimization Granularity

In a typical C++ project setup, it is very hard to ask the compiler to make different optimization tradeoffs for different parts of your program. The best you have is per file granularity on specifying optimization level. This maps poorly to what we want in games. I have a function that has a hot loop, and I want that hot loop _and everything it calls_ to be as optimized as possible. "everything it calls" will be spread out over many different files.

Burst is designed to take as input not your entire program, but a single method in that program: the entrypoint to a hot loop. It will compile that function and everything that it invokes (which is guaranteed to be known: we don't allow virtual functions or function pointers).

Because Burst only operates on a relatively small part of the program, we set optimization level to 11. Burst inlines pretty much everything, making it possible to remove if checks that otherwise would not be removed (because in inlined form we have more information about the arguments of the function).

## Help with common multi threading problems

C++ (nor C#) doesn't do much to help developers to write thread safe code.

Even today, more than a decade since game consumer hardware has >1 cpu, it is very hard to write programs that use multiple core's effectively.

Data races, indeterminism and deadlocks are all challenges that make shipping multi threading code difficult. What we want is features like "make sure that this function and everything that it calls never read or write global state". We want violations of that rule to be compiler errors, not "guidelines we hope all programmers adhere to". Burst gives a compiler error.

We encourage users (and ourselves) to write "jobified" code: splitting up all
data transformations that need to be happen in jobs. Each job is "functional", as in side-effect free. It explicitely specifies its readonly buffers and read/write buffers it operates on. Any other attempt to access other data results in a compiler error.

The job scheduler will guarantee that nobody is writing to your readonly buffer while your job is running. And we'll guarantee that nobody is reading from your read/write buffer while your job is running.

If you schedule a job that violates these rules, you get a runtime error _every time_. Not just in your unlucky race condition case. The error message will explain that you're trying to schedule a job that wants to read from buffer A, but that you already scheduled a job before that will write to A, so if you want to do this, you need to specify that previous job as a dependency.

We find this safety mechanism catches a _lot_ of bugs before they get committed,
resulting in efficient use of all cores. It becomes impossible to code a deadlock or a race condition. Results are guaranteed to be deterministic regardless of how many threads are running, or how many time a thread gets interrupted by some other process.

## Controlling the whole stack

By having control over all these components we can get  benefits by making them be aware of eachother. 
For example, a common case for a vectorization not happening, is that the compiler cannot guarantee that two pointers do not point to the same memory (aliasing). We know two NativeArray's will never alias, because we wrote the collection library, and we can use that knowledge in Burst, so it won't have to give up on an optimization because it's afraid two array pointers might point to the same memory.

Similarly, we wrote the [Unity.Mathmetics](https://github.com/Unity-Technologies/Unity.Mathematics) math library. Burst has intimite knowledge of it. It will (in the future) be able to do accuracy sacrificing optimizations for things like math.sin(). Because to Burst math.sin() is not just any C# method to compile, it will understand the trigonometric properties of sin(), understand that sin(x) == x for small values of x (which Burst might be able to prove), understand it can be replaced by a taylor series expansion for a certain accuracy sacrifice.

## Distinction between engine code and game code dissapears

By writing Unity's runtime code in HPC#, the "engine" and the game are written
in the same language. Runtime systems that we have converted to HPC#, we will distribute as source. Everyone will be able to learn from them, improve them, tailor them. We'll have a level playing field, where nothing is stopping users from writing a better particle system / physics system / renderer than we write.
I expect many people will ([here's a user writing a physics engine](https://forum.unity.com/threads/sources-available-physics-in-pure-ecs.531716/) completely in user code, before we ship a HPC# physics engine). By having our internal development process be much more like our users development process, we'll also feel our users pain more directly,
and we can focus all our efforts into improving a single workflow, instead of two different ones.

## Join us

Many game industry veterans (and non veterans) are unhappy with the status quo and decided to join us to make this a reality (Hi 
[@postgoodism](https://www.twitter.com/postgoodism), 
[@xoofx](https://www.twitter.com/xoofx), 
[@deplinenoise](https://www.twitter.com/deplinenoise), 
[@icetigris](https://www.twitter.com/icetigris), 
[@macton](https://www.twitter.com/macton), 
[@vengefularia](https://www.twitter.com/vengefularia),
[@bmcnett](https://www.twitter.com/bmcnett)). If changing the status quo in langauge tools for game development is something you would also love to work on, we'd love to hear from you. Ping me [@lucasmeijer](https://www.twitter.com/lucasmeijer) or Mike Acton [@macton](https://www.twitter.com/macton) if you're interested.

The future isn't going to build itself.

There are no comments on this website, but you can say hi on twitter [@lucasmeijer](https://www.twitter.com/lucasmeijer)


Footnotes:

- Mike Acton did [a CppCon keynote](https://youtu.be/rX0ItVEVjHc) that does a good job of illustrating the gap between what game programmers need and what C++ offers: 

- [Jonathan Blow](https://www.twitter.com/Jonathan_Blow) is working on a interesting language + compiler called Jai, sharing some motivation with unity: if I'm going to spend another 20 years making games, let's make sure we do it with tools I love using.