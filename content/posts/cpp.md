---  
title: "Games, C++ and Unity"  
date: 2018-12-28T17:22:10+03:00    
comments: false
---

A lot has been said and written lately about the game industry's "C++ is not changing into the thing we need" feelings. Valid criticism on various things that make C++ not a great language for games (or at all), and valid counter criticism of "well why dont you guys participate in the open-for-everyone process of designing C++ instead of bitching from the sidelines".

Let's talk about the place C++ will have at Unity in the future:

Unity and game developers' problem at the end of the day is that they need to provide an executable with instructions the target processor can understand, that when executed will run the game.

For the performance cricital part of our code, we know what we want the final instructions to be.
We just want an easy way to describe our logic in a reasonable way, and then trust and verify that the
generated instructions are the ones we want.

Common problems trying to use C++ for this tasks are

- I want my loop to be vectorized, but a million things can happen that might make the compiler not vectorize it. It might be vectorized today, but not tomorrow if a new seemingly innocent change happens.

- convincing my c/c++ compiler to vectorize my code at all is hard.


We took a step back and figured: wow it's kind of crazy the only popular way to generate machine instructions
somewhat comfortably is C/C++. (There's Go and Rust these days too, which is great, this space can use some competition). We decided to make our own "reasonably comfortable way to generate machine code", that checks all the boxes that we care about. We could spend a lot of energy trying to bend the C++ design train a little bit more in a direction it would work a little bit better for us, but we'd much rather spend that energy on a toolchain where we can do all of the design, and that we design exactly for the problem that
game developers have.

What boxes do we care about?

- Performance is correctness. I should be able to say "if this loop for some reason doesn't vectorize, that should be a compiler error, not a 'oh code is now just 8x slower but it still produces correct values, no biggy!'"

- We should have a nice iteration loop where I can easily see the machine code that is generated. The machine code "viewer" should do a good job at teaching/explaining what all these machine instructions do.

- Cross-architecture. The input code I write should not have to be different for when I target iOS than when I target xbox. (this sounds like a no brainer, but after pulling your hair out getting a C++ compiler to reliably generate the instructions you want, it's very common to just tableflip, and write the instructions you want in assembler and be done with it)

- Safety. most game developers don't have safety very high on their priority list, but we think that the fact that it's really hard to have memory corruption in Unity has been one of its killer features. There should be a mode in which we can run this code that will give us a clear error with a great error message if I read/write out of bounds or dereference null

Ok, so now we know what things we care about, next step we need to decide on what the input language for this
machinecode generator is. What options do we have:

- Custom language
- Some adaption/subset of C or C++
- Subset of C#

Say what C#? For our most performance critical inner loops?

Yes. C# is a very natural choice that comes with a lot of nice benefits for Unity:

- It's the language our users already use today
- Has great IDE tooling that C++ programmers often have no idea even was possible
- A C#->intermediate IL compiler already exists, and we can just use it instead of having to write our own
- We have a lot of experience modifying intermediate-IL, so it's easy to do codegen and postprocessing on the actual program

I quite enjoy writing code in C# myself. However, C# as a whole is not an amazing language when it comes to performance. While the c# language team, library team, and runtime team have been making great progress in the last two years, we're still looking at a language where you have no control over where/how your data is laid our, while control over how your data is laid out, and how your memory is allocated is exactly what we need. On top of that the standard library is oriented around "objects on the heap", and "objects having pointer references to other objects".

That said, if we give up on the most of the standard library, (bye Linq, StringFormatter, List<T>, Dictionary), disallow allocations (=no classes, only structs), dissalow virtual calls and non-constrained interface invocations, and add a few new containers that you are allowed to use (NativeArray<T> and friends) the remaining pieces of the C# language are looking pretty good. Remember this is only for your performance critical code. Here's an example from our mega city demo:

This subset lets us comfortably do everything we need. Because it's a valid subset of c#, we can also run it as regular c# getting safety, great errors, debugger support, and the ability to easily move code out of and into the performance critical part of our game. We often refer to this subset as HPC#.

## Where are we today?

We've built this code generator / compiler, and it's called Burst. It ships with Unity2018.3. We obviously
have a lot of things left to do on it, but we're already very happy with it today. Performance is often better than C++, rarely worse. But comparing performance is not exactly the right thing to do. What matters is what you had to do to get that performance. As an example we took some of our c++ culling code, and ported it to burst. performance was the same, but the c++ version had to do incredible gymnastics to convince our c++ compilers to actually vectorize. The burst version was about 6x smaller, running slightly faster.

To be honest, the whole "you should move your most performance critical code to c#" story also didn't result in everybody internally at Unity immediately buying it. For most of us it feels like "you're closer to the metal" when you use C++. But at Unity it's not true anymore. When we use c# we have complete control over the entire process from compilation down to machine code generation, and if there's something we don't like, we just go in and fix it. No comittee to convince of the value of our usecase, or to balance against other concerns.

We will slowly but surely port every piece of performance critical code that we have in C++ to HPC#.
It's easier to get the performance we want, harder to write bugs, and easier to work with.

Unity has a lot of different users. Some can enumerate the entire arm64 instruction set from memory, others
are just starting and excited that they can already use their creativity and create things before getting a phd in computer science.

All users will benefit, by virtue of unity's runtime code, and their asset store's packages runtime code being converted more and more to HPC#. Advance users will benefit ontop of that by being able to also
write their own high performance code in HPC#.



## Where are we going

I am very excited that we have a path forward that we think we can make amazing, in which we have full control. By having all these components in house, we can get extra benefits because we can make them be aware
of eachother. For example, a common case for a vectorization not happening, is that the compiler cannot guarantee that to pointers do not point to the same memory (aliasing). We know two NativeArray<<T>>'s will
never alias, because we wrote the collection library, and we can use that knowledge in the burst compiler,
so it won't have to give up on an optimization because it doesn't have enough information about the input data.

