# Gammakit

Gammakit is a toy programming language I'm making (in an unstable design/development state, possibly permanently) loosely inspired by Game Maker Language (herein "gml") and a couple concepts I picked up from partially reverse engineering cutscene scripting systems used in adventure games.

Gammakit is a lexically scoped, dynamically typed, imperative programming language with support for first-class functions, generators (semicoroutines), and self-modifying code. Variables do not have reference semantics; lambdas can only capture by value.

Pointer-like semantics(*) are applied to instances of user-defined object types, where those instances must be created and freed explicitly by the user with functions dedicated to doing so; the assumption is that the programmer sees instances as belonging to the "world", rather than any particular chunk of code.

(*) as in, you can mutate the same data from multiple places based on a shared bit of stored data, without going through global scope

## Previous attempts

The first attempt at making a gml-inspired programming language was called "notgml". It was a buggy, horribly incomplete mess, confined to two C++ source files totalling 4500 lines.

The second attempt was the first time it bore the name "gammakit". It was written in python 3. It did not reach a usable state before being abandoned.

The third and current attempt is written in rust. The codebase is approximately 6000 lines of real code. I was aiming for 4000, but adding desired language features pushed it above this limit. It is certainly never going to break past 10k lines unless I start adding crazy compiler optimizations or anything like that.

## Design hiccups

Gammakit originally emulated lexical scope at runtime, with the (ad-hoc, bytecode-based) interpreter working entirely with strings for all variable names. When a variable was accessed, it would search up a stack of scopes looking for the first reference to a variable.

This also meant that you didn't know whether a piece of code was referencing unknown identifiers or not until it ran. On the other hand, it enabled something called "subroutine-like functions", which acted similarly to C macros in that they could access the visibility context that they were "called" in, but different in that they had to be structured exactly like functions (and indeed looked just like normal functions to the interpreter, just with a single flag flipped).

First of all, the way strings were packed into the bytecode made it necessary to pull them out one character at a time, because they were null-terminated. This was very slow. I eventually made it faster by searching for the null terminator and then grabbing that whole slice out of the bytecode, but quickly scrapped the whole idea and started using unique string indexes, with a mapping from indexes to strings stored next to the bytecode.

This wasn't a good enough speed increase, so I ended up scrapping the lexical scope emulation and made it so that the compiler and interpreter have a model where "function call frames have a stack of variables" and "references to variables just say where on the variable stack the variable is". This is not to be confused with the evaluation stack, which is what normal expression evaluations etc. work off of (rather than registers). This is much cleaner than having a stack of scopes at runtime anyway. Rest in peace, subroutine-like functions. You will be missed. Unironically.

(I have ideas about, basically, reimplementing subroutine-like functions by doing the equivalent of "monomorphizing" for them at compile time, but for variable visibility information, rather than for type information; but this requires that the compiler be aware of the value of the subroutine-like function's identifier at compile time, so for sanity's sake I would have to make them be global in some way. I haven't decided what to do with this idea yet.)

## Design hiccups part two

Gammakit has a few operations that Do Something to a variable, rather than just storing a value in it or reading its value out onto the evaluation stack. These are things like +=, invoking a generator (more on that later), or using an "arrow" binding (described later; they can modify the value of a variable if called under one).

The first way of implementing these was way back when lexical scope was emulated at runtime. When a variable had to be modified (not just overwritten), it would be found, have its value read out, have that value fed into something to get some other value out, then find it again but this time to insert a new value in instead of return its current value. The old implementation of variable access did not have any way for the interpreter to grab a mutable reference to a variable, so anything that had to mutate a variable in-place had to work like this instead.

I eventually realized that this was a bad idea and slow, and (before moving to static, compile-time lexical scope) made variable accesses return an Rc<RefCell<...>> to a value. (In reality, it was a struct, so that there could be read-only and non-reference states, to support read-only identifiers and also using the same type for certain non-variable-related aspects of how the interpreter handles values.)

Much later, concerned by the probable performance overhead of using RefCell, I moved to a way of returning references that utilized lifetimes to keep track of reference lifetime instead of smart pointers. It took a while to get working, but it works quite well, and I can't see any obvious ways to improve it any more.

## Basic design ideas

Gammakit is highly restrained by being, basically, a eulogy to gml. GML is basically a "bad" programming language, but it's bad in ways that are good for game logic programming, to a limited degree.

### Thanks, gml

First of all, in gml, there are no reference semantics (except for instances), and memory management is entirely automatic (once again except for instances), which eliminates multiple whole entire classes of gotcha!s that most programming languages have when non-programmers copy-paste cruddy logic together in them.

Second of all, in gml, there's an interesting quirk to the way scope works. Rather than having lexical or dynamic scope, variables can either belong to a "script" or to an "instance". This is actually bad and confusing (you can't have a script variable in one block, while referencing an instance variable in an unrelated block somewhere else in the same script), but it's similar to how C++ class/struct methods can access their object's variables without any prefixing (though prefixing is still possible). This makes it less verbose to do things like "add this gravity constant to my vspeed variable every frame", so new game logic programmers don't have to go through the process of learning that they need to prefix object attributes with "this." or "self." to use them. This is, admittedly, very minor, but it becomes more interesting with the following ponit.

Third of all, gml has "with". "with" is, in one way, similar to the maligned javascript "with". The first difference is that it's less bad, because gml doesn't throw object variables into the same identifier pool as the metadata that describes how objects work (also: not having variable-level reference semantics). The second difference is that **gml's "with" is a loop**. The following code loops over all extant instances of the object "Character" and decreases their hp variable (if they have one (they do; it's an implied object variable in gml)) by 10:

```gml
with(Character)
{
    hp -= 10;
}
```

That's right, gml stores lists of every instance that exists of every type and makes it possible to just iterate through them all. If scripting languages are all about syntactical sugar, this is basically a game logic programmer's monkfruit. Imagine being able to just go, "okay, for this platforming level's gimmicky trap let's just take EVERY SINGLE BULLET THAT EXISTS IN THE ENTIRE GAME WORLD and just point them directly at the player character. No need to add a new list of bullet instances anywhere or search through component tags or fuss around with object data visibility or add a new method, I just write a couple lines of code and go ```direction = point_direction(x, y, global.myself.x, global.myself.y);``` in the middle of it". (gml also has a few special variables that invoke special hidden functionality when modified; the "speed" and "direction" variables modify the "hspeed" and "vspeed" variables when modified. Gammakit does not attempt to provide this sort of functionality.)

These three design ideas - no reference semantics, implicit access of instance scope (in code defined under them), and 'with' as a loop (and also a provider of implicit access of instance scope) - provide the basic compass for Gammakit design direction.

### Making cutscene scripting painless

There's one more design idea I want to specify: the desire to write things like cutscenes or AI scripts in an arbitrarily modified version of the normal game logic programming language.

This requires two things.

The first is runtime code generation/modification. The motivation for this is having "special game logic things" interspersed with normal code, like "take this unit, make them walk over there. yield execution to the rest of the game until they get there,  then come back to me", but without being that verbose. If the game logic programmer can provide a modified version of the gammakit grammar to a parsing function, they can add arbitrary forms of statements that best suit what they want to do with their "special game logic things". A cutscene designer could add a statement type that just consists of a single screen, indicating that there's a change in the onscreen text, which then yields to the rest of the game for a while, etc. rather than doing it manually.

And code generation might as well be dynamic, given that I'm designing a primarily interpreted programming language.

The second thing it requires is generators or semicoroutines. This is so that "yield execution to the rest of the game" makes conceptual sense. You can do things like that systems languages like C using threads (in fact, misusing threads (i.e. no multicore safety) in the process of doing this resulted in compatibility problems for early PC games running on modern multicore hardware), but gammakit is *strictly* single-threaded, so this is not an option. Additionally, gammakit's semantics and interpreter also do not work in a way where interrupts (the things that make "reentrancy" important in C) make conceptual sense. Finally, implementing some sort of asynchronous execution system would basically be a *more difficult and less fitting* version of implementing generators/semicoroutines, so I just went with implementing generators/semicoroutines.

## The basics of gammakit

TODO

### Types

### User-defined functions

### Lambdas

### Globals

## Objects and instances

## Code generation

## Generators

## Bindings

## Embedding

The primary usage scenario for gammakit is being embedded inside of game engines. As a result, gammakit's built-in suite of bindings is quite limited. The only IO mechanism provided is printing to the console.

Gammakit is designed as a library where the parent application can create and run any number of interpreters in any way that it wants, even going as far as being able to step them instruction-by-instruction. It's also possible to insert your own bindings (obviously) as lambdas that carry in, for example, mutable smart pointers or mutexes, so that bindings can Do Things. This is exactly what the toy game engine "magmakit" does, providing things like a window, keyboard/mouse input, and basic 2d rendering.

Gammakit suspiciously lacks a module system. Instead of having modules and having them jump from one to the other, gammakit's interpreter can have new code loaded into it arbitrarily. This is how magmakit's framerate limiter and input updates work. Rather than being called on in gammakit itself, they silently happen between drawing and stepping, in a loop.

But the embedding application can go about this in any way that it wants, such as having a binding that cycles the rendering system, a binding that cycles the framerate limiter, and a binding that cycles OS input updates, calling them at the tail end of a single massive while() loop. The point being that the embedding application can do whatever it wants; different approaches make sense for different applications, either for design reasons or because of library/OS constraints.
