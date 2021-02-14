# What I’ve been working on, and what’s coming up
    
*Originally published on [2014-06-25](https://6guts.wordpress.com/2014/06/25/what-ive-been-working-on-and-whats-coming-up/) by Jonathan Worthington.*

It’s been a little while since I wrote an update here, and with a spare moment post-teaching I figured it was a nice time to write one. I was lucky enough to end up being sent to the arctic (or, near enough) for this week’s work, meaning I’m writing this at 11pm, sat outside – and it’s decidedly still very light. It just doesn’t do dark at this time of year in these parts. Delightful.

### Asynchronous Bits

In MoarVM and Rakudo 2014.05, basic support for asynchronous sockets landed. By now, they have also been ported to the JVM backend. In 2014.06, there were various improvements – especially with regard to cancellation and fixing a nasty race condition. Along the way, I also taught MoarVM about asynchronous timers, bringing time-based scheduling up to feature parity with the JVM backend. I then went a step further and added basic file watching support and some signals support; these two sadly didn’t yet make it to the JVM backend. On signals, I saw a cute lightning talk by lizmat using signal handling and phasers in loops to arrange Ctrl+C to exit a program once a loop had completed its current iteration.

While things basically work, they are not yet as stable as they need to be – as those folks implementing multi-threaded asynchronous web servers and then punishing them with various load-testing tools are discovering. So I’ll be working in the next weeks on hunting down the various bugs here. And once it’s stable, I may look into optimizations, depending on if they’re needed.

### Various Rakudo fixes an optimizations

I’ve also done various assorted optimizations and fixes in Rakudo. They’re all over the map: optimizing for `1..100000 { }` style loops into cheaper `while` loops, dealing with various small bugs reported in the ticket system, implementing a remaining missing form of colonpair syntax (`:42nd` meaning `nd => 42`), implementing `Supply.on_demand` for creating supplies out of…well…most things, optimizing `push` / `unshift` of single items…it goes on. I’ll keep trucking away at these sorts of things over the coming months; there’s some bugs I really want to nail.

### MoarVM’s dynamic optimizer

A bunch of my recent and current work revolves around MoarVM’s dynamic bytecode optimizer, known as “spesh” (because its primary – though not only – strategy is to specialize bytecode by type). Spesh [first appeared](https://6guts.wordpress.com/2014/04/12/optimization-concurrency-and-moar/) in the 2014.04 release of MoarVM. Since then, it’s improved in numerous ways. It now has a logging phase, where it gathers extra type information at a number of places in the code. After this, it checks if the type information is stable – which is often the case, as most potentially polymorphic code is monomorphic (or put another way, dynamic languages are mostly just eventually-static). Provided we did get consistent types recorded, then guard clauses are inserted (which cheaply check we really did get the expected type, and if not triggering deoptimization – falling back to the safe but slower bytecode). The types can them be assumed by the code that follows, allowing a bunch of optimizations to code that the initial specializer just couldn’t do much with.

Another important optimization spesh learned was optimizing dispatch based on the type information. By the time the code gets hot enough to specialize, multiple dispatch caches are primed. These, in combination with type information, are used to resolve many multiple dispatches, meaning that they become as cheap as single dispatches. Furthermore, if the callee has also been specialized – which is likely – then we can pick the appropriate specialization candidate right off, eliminating a bunch of duplicate guard checks. Everything described so far was on 2014.05.

So, what about 2014.06? Well, the 2014.05 work on dispatch – working out exactly what code we’re going to be running – was really paving the way for a much more significant optimization: inlining. 2014.06 thus brought basic support for inlining. It is mostly only capable of NQP code at the moment, but by 2014.07 it’ll be handling inlining the majority of basic ops in Rakudo that the static optimizer can’t already nail. Implementing inlining was tricky in places. I decided to go straight for the jugular and support multi-level inlining – that is, inlining things that also inline things. There are a bunch of places in Raku this will be useful; for example, `?$some_int` compiles to `prefix:<?>($some_int)`, which in turn calls `$some_int.Bool`. That is implemented as `nqp::p6bool(nqp::bool_I(self))`. With inlining, we’ll be able to flatten away those calls. In fact, we’re just a small number of patches off that very case actually being handled.

The other thing that made it tricky to implementing inlining is that spesh is a speculative optimizer. It looks at what types it’s seeing, and optimizes assuming it will always get those types. Those optimizations include inlining. So, what if we’re inlining a couple of levels deep, are inside one of the inlined bits of code, and something happens that invalidates our assumptions? This triggers de-optimization. However, in the case of inlining, it has to go back and actually create the stack frames that it elided creating thanks to having applied inlining. This was a bit fiddly, but it’s done and was part of the 2014.06 release.

Another small but significant thing in 2014.06 is that we started optimizing some calls involving named parameters to pull the parameters out of the caller’s callsite by index. When the optimization applies, it saves doing any string comparisons whatsoever when handling named parameters.

So 2014.07 will make inlining able to cope better with Rakudo’s generated code, but anything else? Well, yes: I’m also working on OSR (On Stack Replacement). One problem today is that if the main body of the program is a hot loop doing thousands of iterations, we never actually get to specialize the loop code (because we only enter the code once, and so far it is repeated calls to a body of code that triggers optimization). This is especially an issue in benchmarks, but can certainly show up in real-life code too. OSR will allow us to detect such a hot looop exists in unoptimized code, go off and optimize it, and then replace the running code with the optimized version. Those following closely might wonder if this isn’t just a kind of inverse de-optimization, and that is exactly how I’ll implement it: just use the deopt table backwards to work out where to shove the program counter. Last but not least, I also plan to work on a range of optimizations to generic code written in roles, to take away the genericity as part of specialization. Given the grammar engine uses roles in various places, this should be a healthy optimization for parsing.

### JIT Compilation for MoarVM

I’m not actually implementing this one; rather, I’m mentoring *brrt*++, who is working on it for his Google Summer Of Code project. My work on spesh was in no small part to enable a good JIT. Many of the optimizations that spesh does turn expensive operations with various checks into cheap operations that just go and grab or store data, and thus should be nicely expressible in machine code. The JIT that brrt is working on goes from the graph produced by spesh. It “just” turns the graph into machine code, rather than improved bytecode. Of course, that’s still a good bit of work, especially given de-optimization has to be factored into the whole thing too. Still, progress is good, and I expect the 2014.08 MoarVM release will include the fruits of brrt’s hard work.