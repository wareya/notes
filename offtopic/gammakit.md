# Gammakit

Gammakit is a toy programming language I'm making (in an unstable design/development state, possibly permanently) loosely inspired by Game Maker Language (herein "gml") and a couple concepts I picked up from partially reverse engineering cutscene scripting systems used in adventure games.

Gammakit is a lexically scoped, dynamically typed, imperative programming language with support for first-class functions, generators (semicoroutines), and self-modifying code. Variables do not have reference semantics; lambdas can only capture by value. Pointer-like semantics are applied to user-defined object instances, which must be created and freed explicitly by the user with special functions; the assumption is that the programmer sees these as belonging to the "world", rather than any particular chunk of code.

## Previous attempts

The first attempt at making a gml-inspired programming language was called "notgml". It was a buggy, horribly incomplete mess, confined to two C++ source files totalling 4500 lines.

The second attempt was the first time it bore the name "gammakit". It was written in python 3. It did not reach a usable state before being abandoned.

The third and current attempt is written in rust. The codebase is approximately 6000 lines of real code. I was aiming for 4000, but adding desired language features pushed it above this limit. It is certainly never going to break past 10k lines unless I start adding crazy compiler optimizations or anything like that.

## Design hiccups

Gammakit originally emulated lexical scope at runtime, with the (ad-hoc, bytecode-based) interpreter working entirely with strings for all variable names. When a variable was accessed, it would search up a stack of scopes looking for the first reference to a variable.

This also meant that you didn't know whether a piece of code was referencing unknown identifiers or not until it ran. On the other hand, it enabled something called "subroutine-like functions", which acted similarly to C macros in that they could access the visibility context that they were "called" in, but different in that they had to be structured exactly like functions (and indeed looked just like normal functions to the interpreter, just with a single flag flipped).

First of all, the way strings were packed into the bytecode made it necessary to pull them out one character at a time, because they were null-terminated. This was very slow. I eventually made it faster by searching for the null terminator and then grabbing that whole slice out of the bytecode, but quickly scrapped the whole idea and started using unique string indexes, with a mapping from indexes to strings stored next to the bytecode.

This wasn't a good enough speed increase, so I ended up scrapping the lexical scope emulation and made it so that the compiler and interpreter have a model where "function call frames have a stack of variables" and "references to variables just say where on the variable stack the variable is". This is not to be confused with the evaluation stack, which is what normal expression evaluations etc. work off of (rather than registers). This is much cleaner than having a stack of scopes at runtime anyway. Rest in peace, subroutine-like functions. You will be missed. Unironically.

(I have ideas about, basically, reimplementing subroutine-like functions by "monomorphizing" variable visibility information at compile time, but this requires that the compiler be aware of the value of the subroutine-like function's identifier at compile time, so for sanity's sake I would have to make them be global in some way. I haven't decided what to do with this idea yet.)

## Design hiccups part two

Gammakit has a few operations that Do Something to a variable, rather than just storing a value in it or reading its value out onto the evaluation stack. These are things like +=, invoking a generator (more on that later), or using an "arrow" binding (which can modify the value of a variable if called under one).

The first way of implementing these was way back when lexical scope was emulated at runtime. When a variable had to be modified (not just overwritten), it would be found, have its value read out, have that value fed into something to get some other value out, then find it again but this time to insert a new value in instead of return its current value. The old implementation of variable access did not have any way for the interpreter to grab a mutable reference to a variable, so anything that had to mutate a variable in-place had to work like this instead.

I eventually realized that this was a bad idea and slow, and (before moving to static, compile-time lexical scope) made variable accesses return an Rc<RefCell<...>> to a value. (In reality, it was a struct, so that there could be read-only and non-reference states, to support read-only identifiers.)

Much later, concerned by the probable performance overhead of using RefCell, I moved to a way of returning references that utilized lifetimes to keep track of reference lifetime instead. It took a while to get working, but it works quite well, and I can't see any obvious ways to improve it any more.

## Basic design ideas

TODO
