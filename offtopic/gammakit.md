Regardless of how much this article sounds like a thought experiment or piece of satire masquerading as a design retrospective, the projects described in this article *actually exist*, and this article is not fiction or comedy. [Gammakit](https://github.com/wareya/gammakit/) and [magmakit](https://github.com/wareya/magmakit/). I am not an academic and I have not studied programming language design formally. I just did this for fun.

This article is mostly done, but has not been edited, and might be missing parts I forgot about. Gammakit is also in a state of design flux, so this article might change dramatically or fall slightly out of date (though I will do my best to keep it in sync with gammakit's current development status).

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

Gammakit originally emulated static lexical scope at runtime, with the (ad-hoc, bytecode-based) interpreter working entirely with strings for all variable names. When a variable was accessed, it would search up a stack of scopes looking for the first reference to a variable.

This also meant that you didn't know whether a piece of code was referencing unknown identifiers or not until it ran. On the other hand, it enabled something called "subroutine-like functions", which acted similarly to C macros in that they could access the visibility context that they were "called" in, but different in that they had to be structured exactly like functions (and indeed looked just like normal functions to the interpreter, just with a single flag flipped).

First of all, the way strings were packed into the bytecode made it necessary to pull them out one character at a time, because they were null-terminated. This was very slow. I eventually made it faster by searching for the null terminator and then grabbing that whole slice out of the bytecode, but quickly scrapped the whole idea and started using unique string indexes, with a mapping from indexes to strings stored next to the bytecode.

This wasn't a good enough speed increase, so I ended up scrapping the lexical scope emulation and made it so that the compiler and interpreter have a model where "function call frames have a stack of variables" and "references to variables just say where on the variable stack the variable is". This is not to be confused with the evaluation stack, which is what normal expression evaluations etc. work off of (rather than registers). This is much cleaner than having a stack of scopes at runtime anyway. Rest in peace, subroutine-like functions. You will be missed. Unironically.

(I have ideas about, basically, reimplementing subroutine-like functions by doing the equivalent of "monomorphizing" for them at compile time, but for variable visibility information, rather than for type information; but this requires that the compiler be aware of the value of the subroutine-like function's identifier at compile time, so for sanity's sake I would have to make them be global in some way. I haven't decided what to do with this idea yet.)

## Design hiccups part two

Gammakit has a few operations that Do Something to a variable, rather than just storing a value in it or reading its value out onto the evaluation stack. These are things like +=, invoking a generator (more on that later), or using an "arrow" binding (described later; they can modify the value of a variable if called under one).

The first way of implementing these was way back when static lexical scope was emulated at runtime. When a variable had to be modified (not just overwritten), it would be found, have its value read out, have that value fed into something to get some other value out, then find it again but this time to insert a new value in instead of return its current value. The old implementation of variable access did not have any way for the interpreter to grab a mutable reference to a variable, so anything that had to mutate a variable in-place had to work like this instead.

I eventually realized that this was a bad idea and slow, and (before moving to static, compile-time lexical scope) made variable accesses return an Rc<RefCell<...>> to a value. (In reality, it was a struct, so that there could be read-only and non-reference states, to support read-only identifiers and also using the same type for certain non-variable-related aspects of how the interpreter handles values.)

Much later, concerned by the probable performance overhead of using RefCell, I moved to a way of returning references that utilized lifetimes to keep track of reference lifetime instead of smart pointers. It took a while to get working, but it works quite well, and I can't see any obvious ways to improve it any more.

## Basic design ideas

Gammakit is highly restrained by being, basically, a eulogy to gml. GML is basically a "bad" programming language, but it's bad in ways that are good for game logic programming, to a limited degree.

### Thanks, gml

First of all, in gml, there are no reference semantics (except for instances), and memory management is entirely automatic (once again except for instances), which eliminates multiple whole entire classes of gotcha!s that most programming languages have when non-programmers copy-paste cruddy logic together in them.

Second of all, in gml, there's an interesting quirk to the way scope works. Rather than having lexical or dynamic scope, variables can either belong to a "script" or to an "instance". This is actually bad and confusing (you can't have a script variable in one block, while referencing an instance variable in an unrelated block somewhere else in the same script), but it's similar to how C++ class/struct methods can access their object's variables without any prefixing (though prefixing is still possible). This makes it less verbose to do things like "add this gravity constant to my vspeed variable every frame", so new game logic programmers don't have to go through the process of learning that they need to prefix object attributes with "this." or "self." to use them. This is, admittedly, very minor, but it becomes more interesting with the following point.

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

The second thing it requires is generators or semicoroutines. This is so that "yield execution to the rest of the game" makes conceptual sense. You can do things like that systems languages like C using threads (in fact, misusing threads (i.e. no multicore safety) in the process of doing this resulted in compatibility problems for early PC games running on modern multicore hardware), but gammakit is *strictly* single-threaded, so this is not an option. Additionally, gammakit's semantics and interpreter also do not work in a way where interrupts (the things other than threads that make "reentrancy" important in C) make conceptual sense. Finally, implementing some sort of asynchronous execution system would basically be a *more difficult and less fitting* version of implementing generators/semicoroutines, so I just went with implementing generators/semicoroutines.

## The basics of gammakit

Gammakit is fundamentally a vaguely-C-like imperative language, despite how little it actually has in common with real C. It has the same basic kind of control flow mechanisms and the same general model for how basic algorithms are constructed. All of the specifics are quite different, though, and the set of high-level programming features available to the programmer is completely different.

### Types and scope

Gammakit is lexically scoped and dynamically typed. A few things stick out, though: compilation is (currently) single-pass, so functions can't access global variables declared below them, and functions can't access the lexical scope they're defined in (more on this later). Object types also cannot be accessed before they are declared, though this is less of a problem than it is in C/C++ because all instances have the same type, "instance identifier".

The rudimentary types in gammakit are:

* Number (f64)
* String
* Array
* Dict
* Set
* Function (binding or user-defined)
* Generator state
* Instance (identifier; carries pointer-like semantics)
* Object
* Custom (not used by gammakit itself; available for parent application bindings to pass data into and out from gammakit)

There's also a hidden non-assignable type dedicated to temporarily holding data concerning "arrow bindings" (more on these later, again). This might be considered an implementation detail. Attempting to assign the value of an arrow binding (rather than the result of calling it) to a variable, even indirectly (such as through a function call or by inserting it into an array/dict), is unsupported behavior (the interpreter currently throws an error if you try doing so).

True and false are represented by the numbers 1.0 and 0.0.

Dicts can only have numbers or strings as keys. Sets can only have numbers as keys.

Any variable can hold a value of any assignable type.

Once again, gammakit does not have variable-level reference semantics. In order to access a variable from multiple places, it must be stored in an instance.

Coming off of the lack of reference semantics once again, when a generator state value is assigned to a new variable, it is *completely cloned*. This means that you can fork generators mid-execution (especially because you can "tick" generators a single yield at a time). If you want to access the same generator from multiple places, it must be placed inside of an instance.

While we're here, gammakit also has array, set, and dict literals, unlike old versions of gml:

```gml
var test_array = ["asdf", 0, ["test", 0]];
var test_dict = {"asdf" : 1, 0 : "a"};
var test_set = set {"asdf", 0};
```

#### Operators

Has two basic types of general-purpose operator, as in the kind that you would use for numerical things.

The main binary arithmetic operators (`+`, `-`, `*`, `/`, and `%`) also have binary statement versions (e.g. `x += 5;`).

Warning: a huge gotcha! that's held over from gml: the truthiness of numbers is defined as "n >= 0.5 means true, otherwise means false". Generators truth-test as whether they have finalized. Instance identifiers will truth-test as whether instance still exists in the future; they currently truth-test as true. Everything else currently truth-tests as true; I will probably change this to false, but I'm open to being convinced otherwise, or even to being convinced to make strings, arrays, sets, and dicts truth-test as whether or not they have anything in them, but I do not currently plan to do so.

In addition to the following, gammakit also has a ternary operator, defined syntactically such that each part must be wrapped in parens like `(a) ? (b) : (c)` (todo: male elvis operator compile properly).

##### Binary

* `&&`/`and`, `||`/`or` : Basic boolean "and" and "or". Operate on the truthiness of numerical experssions. Have short-circuiting semantics. There are no other short-circuiting semantics anywhere in gammakit. The preferred representations are `and` and `or`, not `&&` and `||`, though both are supported.
* `==`, `!=` : Equality and inequality operations. Operate meaningfully on numbers, strings, arrays, dicts, sets, functions, instances, and "custom" values. All other types return "false" for equality, and "true" for inequality. Number equality is defined in exactly the same way as rust's `==` operator on two f64 values. String equality is defined in exactly the same way as rust's `==` operator on two &str values. Array equality is defined trivially. Dict equality is defined based on containing the same mappings; insertion order does not matter. Same with sets, except with values, not mappings.
* `<=`, `>=`, `<`, `>` : Ordered inequality operations. Operate only on numbers and strings. Defined the same way as the equivalent rust operators. `<` and `>` return false for all other types. `<=` and `>=` return the result of `==` for all other types.
* `+`, `-`, `*`, `/`, `%` : Arithmetic operations. Operate on numbers. Defined the same way as the equivalent rust operators, EXCEPT for %, which performs euclidean modulo. Random exception: `+` and `*` are defined for strings; `+` performs concatenation between two strings and `*` can be used to repeat a string (left only) a number of times (right only; rounded down to an integer).

Binary operator precedence, among all other precedence-related things, is defined declaratively in gammakit's grammar: https://github.com/wareya/gammakit/blob/22006a9ea3b1b6d6b91f180b66fb684c661243b9/src/defaultgrammar.txt#L106

(The binary operators are defined as right-recursive in the grammar, but they are rotated to be left-recursive in the final AST. They have left-recursive semantics.)

##### Unary (prefix)

The only unary operations are `+` (positive; does nothing to numbers; throws an error for all other types), `-` (negative; flips the sign of a number; throws an error for all other types), and `!` (logical negation; truth-tests an expression, then evaluates to the opposite of that truthiness). Unary operations are placed to the left of the expression they modify.

### Statements

"Statement" is a loaded term. Most language constructs are statements. If they're not, they're expressions. Or part of an expression. At least in the imperative world that gammakit inhabits.

Variable declarations are statements. Variables can be declared anywhere, and you can declare multiple variables on the same line, and set their values. This does not work the same way as it does in C. Each variable gets a unique value, like so:

`var x = 10, y = 3, name = "fighter";`

If a variable does not have a value assigned to it upon declaration, it contains the number 0.0.

Variables can be reassigned in the following ways:

```gml
x = 10;
x += 10; // and the other arithmetic operations
x++; // equivalent to x += 1;
x--; // equivalent to x -= 1;
```

Yes, gammakit has ++ and -- statements. No, they are not expressions, and this means that there's no weird behavioral ordering nastiness going on. These special statements exist to make it so that C programmers don't have to bother themselves with `+= 1` and `-= 1` when writing manual for loops.

There are a few more ways for variables to change value. These include arrow bindings (more on that later) and invoking a generator (once again, more on that later).

Statements can be grouped into blocks with {}, which create a new bubble of lexical scope. Variables can be declared absolutely anywhere within a block, but are only visible below their declaration, and until the block they were declared in closes.

### Control flow

Gammakit has high level control flow, just like any sane programming language, and this is a list of which high level control flow constructs it has. Control flow mechanisms generally do not require {} around their inner blocks like they do in rust (unless, of course, those inner blocks consist of multiple statements, in which case the {} is necessary). However, even if there is no {}, it's still a "block" with its own bubble of lexical scope.

#### If, else, while

Self explanatory. Break and continue work properly for while loops, too.

#### For (manual)

These compile down to while loops with a bit of a prelude. Yes, initialization and scope and break and continue are handled correctly and interact with the initialization and loop statement and conditional properly and everything executes in the same order it executes in in C. Each part of the header (init, cond, loop) can be elided, just like in C. Eliding the cond makes it always evaluate to true, just like in C.

One fun thing that gammakit borrows from game maker is that the "init" and "loop" elements of the header can be *entire blocks*, like so:

```gml
for(var j = 0; j < 10; {j += 1;})
{
    if(j == 4)
        continue;
    print(j);
    if(j == 8)
        break;
}
```

The "var j = 0" statement could be a block too, but this would not be useful unless j were already declared outside of the for loop and you wanted to confuse somebody.

#### For (each)

Foreach loops look like this:

```gml
for(thing in test_array)
    print(thing);
```

where "thing" is implicitly declared and assigned at the header of the loop as being the next element of test_array on each iteration. Foreach loops also work on dicts and sets, but the iteration order is currently undefined (in the future, it will be defined based on insertion order).

Foreach loops also work on generators, but more on that later.

#### Switch (but not your uncle's switch)

Switch statements are superficially similar to the switch statements from C, but their design deviates significantly.

First of all, cases can be arbitrary expressions, not just literal values or constants.

Second of all, there is no fallthrough at all (and thus no need for a "break" or "continue" statement), and having multiple values test for the same case is just done by listing them with `,` (you know what? I lied earlier, this `,` short-circuits too, just like the `and` and `or` operators).

Third of all, each case opens up its own implicit block of scope.

Switches are an exception to the general rule that flow control facilities in gammakit do not require {}. Switches are a flow control facility that *does* require {}. And what's more, it's recommended to not indent the switch's {}, but rather the cases themselves, because those cases are where new scopes open and close, not the switch's {}.

```gml
var x = "test";

def randomtestfunction()
{
    print("this won't run!");
    return 2;
}

switch (x)
{
case 0:
    print("first block");
case 1, "test", randomtestfunction():
    var x = "test2"; // this is a new block, so it doesn't clash with the `var x = "test";` above
    print("second block, this should run");
case "test":
    print("not run");
default:
    print("also not run");
}
```

This basically eliminates all of the bad things about switch statements and makes it possible to use them as a compact/better-structured version of if/else chains rather than exclusively relegating them to only doing things that would be done better with pattern-matching facilities (that gammakit does not have).

#### With

There are two forms of "with" in gammakit. The first is a loop over all extant instances of an object in the world, with instance scope access shenanigans. The second is a block run against a single instance, only with the instance scope access shenanigans. The second form requires an object type annotation; otherwise the compiler cannot know what variables the instance has when doing lexical scope things.

```gml
with(i as Character) // "i" is a variable holding the identifier of an instance
{
    print(x); // this "x" is a property of the instance
    x -= 1;
}

with(Character)
{
    var asdfe4 = 10; // declaring a variable in local lexical scope while in a with statement
    printthing("this is an argument"); // printthing is a method of the Character object type; it's called against whatever instance is the context of the current iteration of the loop
}
```

The type annotation of the non-loop version and the loopy nature of the loop version should, hopefully, reduce any disgust that might come from people who know how bad JS's "with" is.

#### Goto

Unfortunately, gammakit does not have a "goto". I couldn't figure out how to make it work cleanly and safely. I hope that having generators and first-class functions and lambdas makes up for not having it.

### User-defined functions

User defined functions can be declared anywhere that you can make a normal statement and accessing their identifiers works the same way as accessing normal lexically scoped variables. There's also a way to declare functions into the global scope, which is okay because user-defined function identifiers are read-only. More on global functions soon.

Functions return the number 0.0 by default if they fall through their last line.

It's true that the identifiers of user-defined functions are read-only, but you can pass the value contained in those identifiers around freely and store them in variables. This value describes everything there is about the function.

```gml
def rewrite(ast, callback)
{
    ast = callback(ast);
    if(ast["isparent"])
    {
        var max = ast["children"]->len();
        for(var i = 0; i < max; i += 1)
            ast["children"][i] = rewrite(ast["children"][i], callback);
    }
    return ast;
}
```

Functions can access their own name in their own body, allowing for recursion, despite the combination of first-class functions, no reference semantics, and lexical scope making it seem like this should not be possible. The compiler and interpreter work together to make sure that there's always an identifier containing the function's description in scope, with the same name as the function is declared with.

### Lambdas

Lambdas are basically the same as user-defined functions, except that they can capture (by value, to specified variable names) and that they don't have an inherent identifier. The identifier used for recursion in lambdas is, therefore, lambda_self.

```gml
var countdown = [](x)
{
    if(x > 0)
    {
        print(x);
        lambda_self(x-1);
    }
    else
        print("Liftoff!");
};

countdown(10);
```

```gml
var mylambda = [x = "hello, world!", y = "adsf"]()
{
    print(x);
    x = "f";
    print(x);
    {
        var x = "hello, nobody!";
        print(x);
    }
};
mylambda();
mylambda(); // same every time
mylambda();
mylambda();
```

### Globals

There are three forms of global identifier in gammakit. The first is global variables, which must be prefixed with `global.` when accessed. The second is bare global variables, which must be assigned upon declaration and cannot be reassigned or mutated, and must additionally have identifiers that begin with an uppercase ascii letter. The third is global functions, which like normal user-defined functions have immutable identifiers, but can be accesse from anywhere without `global.`, unlike global variables, and do not need their first character to be an uppercase ascii letter.

Note that because instance identifiers themselves are not mutated when you modify a variable in an instance, you can mutate instances through bare global variables.

```gml
globalvar x;

print(global.x);
global.x = 10;
print(global.x);

globaldef testfunc()
{
    print("I was accessed!");
}

def call_testfunc()
{
    testfunc();
}

call_testfunc();
```

```gml
bare globalvar Gravity = 0.06;

bare globalvar MyChar = instance_create(Character);
```

## Objects and instances

Objects have a set of variables and methods which are specified in-place when the object is declared. These can be accessed through an instance identifier with the . operator (which was excluded from the normal binary operators because it does not parse the same way and does not work in a remotely similar way in the compiler either), and for variables, mutated as well.

```gml
obj Tile {
    var sprite;
    var x, y, offsets, bbox;
    def create()
    {
        sprite = S_Tile;
    }
    def init(arg_x, arg_y)
    {
        x = 16+arg_x*32;
        y = 16+arg_y*32;
        
        offsets = [-16, -16, +16, +16];
        bbox = [0,0,0,0];
        
        update_bbox(id); // some global function that modifies id.bbox based on id.x etc
    }
}

def make_tile(x, y)
{
    var tile = instance_create(Tile);
    tile.init(x, y);
}

for(var i = 0; i < 20; i++)
    make_tile(i,5-floor(i/4));
```

Gammakit does not currently have anything resembling inheritance or traits. If stuffing things inside of dicts and passing lambdas or function values around into instances isn't good enough, something resembling inheritance or traits might be added in the future, but I'm still trying to wrap my head around the right way to handle it.

## Code generation

Gammakit provides functions for parsing text into an AST and compiling an AST into a function value (like a lambda). You can also provide your own grammar when parsing text into an AST, if you want to. I *might* add facilities for grabbing the ASTs of the current "file", or specific functions, parts of the program, but this is only a possibility.

The ASTs are themselves just nested dicts and arrays.

```gml
var myf = compile_text("print(\"test\");");
myf();

var myast = parse_text("print(\"toast\");");

var myotherast = myast;

def rewrite(ast, callback)
{
    ast = callback(ast);
    if(ast["isparent"])
    {
        var max = ast["children"]->len();
        for(var i = 0; i < max; i += 1)
            ast["children"][i] = rewrite(ast["children"][i], callback);
    }
    return ast;
}

myotherast = rewrite(myotherast, [](ast)
{
    if(ast["isparent"] and ast["text"] == "string" and ast["children"]->len() > 0)
        if(!ast["children"][0]["isparent"] and !ast["children"][0]->contains("precedence") and ast["children"][0]["text"] == "\"toast\"")
            ast["children"][0]["text"] = "\"not toast\"";
    return ast;
});

var mycode = compile_ast(myast);
mycode();
var myothercode = compile_ast(myotherast);
myothercode();
```

It's also possible to provide your own grammar when parsing a string into an AST.

```gml
// function that takes an AST with a different grammar than gammaikt expects, and replaces the nonstandard parts of it with new standard parts that do something similar
globaldef reprocess_vn_script(ast)
{
    if(ast{isparent})
    {
        var max = ast{children}->len();
        // this AST node follows our nonstandard extension to the gammakit grammar
        if(ast{text} == "statement" and ast{children}->len() == 1 and ast{children}[0]{text} == "string")
        {
            // the string we want to replace the onscreen text with, pulled from the AST
            var text = ast{children}[0]{children}[0]{text};
            // build a new statement block to replace the string in the AST with (using the standard parser because we don't care about our extension anymore)
            // in gammakit's grammar, blocks like this are basically just ordinary statements of their own
            var newstatement = parse_text(
                "{"+
                "   set_current_line("+text+");"+ // a global function that modifies global state
                "   yield;"+
                "}"
            );
            // the root node of a parsed bit of gammakit text as spit out by parse_text is always "program". we want a "statement" instead
            var new_ast = newstatement{children}[0];
            // replace current nonstandard statement AST node with a standard one
            ast = new_ast;
        }
        // apply this function to all children
        for(var i = 0; i < max; i += 1)
        {
            ast{children}[i] = reprocess_vn_script(ast{children}[i]);
        }
    }
    return ast;
}

// text that we display on the screen somewhere else
globalvar display_text = "riptide rush tastes like one of those cheap goo-filled or juice-filled grape-like or citrus-like gummy candies that has a very artificial edge when you first taste it but then the aftertaste kicks in and it's just mildly pleasant all around, even on subsequent sips";
// global function that modifies the onscreen text
globaldef set_current_line(text)
{
    print("running set_current_line");
    global.display_text = text;
}

// load our customized grammar into a string
globalvar grammar = file_load_to_string("data/grammar.txt");
// load our script with nonstandard grammar into a string
var script = file_load_to_string("data/script.gmc");
// parse the script with the custom grammar
var ast = parse_text_with_grammar(script, global.grammar);
// process the AST to be standard gammakit
ast = reprocess_vn_script(ast);
// compile it into a generator that we invoke elsewhere to update the text on screen
globalvar script = compile_ast_generator(ast)();
```

script.gmc:

```gml
"This is the end.";
"The bitter, bitter end.";
```

how grammar.txt differs from the default grammar:

```
statement:
$blankstatement$
$statementlist$
$declaration$ ;
$bareglobaldec$ ;
$condition$
$foreach$
$withstatement$
$withasstatement$
$switch$
$funcdef$
$globalfuncdef$
$objdef$
$binstate$ ;
$unstate$ ;
$instruction$ ;
$funccall$ ;
$invocation_call$ ;
---> $string$ ; (this is added to data/rgammar.txt and doesn't exist in the default grammar)
```

## Generators

Generators will probably be easiest to explain by example:

```gml
generator gentest(i)
{
    while(i > 0)
    {
        yield i;
        i -= 1;
    }
    return "generator has finalized";
}

var test_state = gentest(10);

// prints 10 through 1 then "generator has finalized"
while(test_state)
    print(invoke test_state);

// the generatorstate value inside of the variable "test_state" is now consumed
```

This loop makes the generator produce the numbers 10 through 1, and then finally also produce the string "generator has finalized".

The statement `var test_state = gentest(10);` "initializes" the generator description in "gentest" into a generator state value which is then stored in the variable "test_state".

Generators truth-test as whether or not they have finalized, allowing you to use them as the argument to an "if" or "while".

The "invoke" instruction ticks a generator until its next yield (or return) and produces the yielded or returned value, and if the argument of the "invoke" instruction was a variable, then that variable is updated with the new state of the generator (if it was a non-variable expression then the new generator state is thrown away).

I mentioned this earlier, but generators can be forked just by assigning the value of one variable containing a generator state value to another variable. This copies the generator state value, which includes everything describing the current state of the generator (internally to the interpreter, this description is a function call frame).

Generators can also be fed through for (each) loops.

```gml
test_state = gentest(10);

// this copies the generatorstate inside of test_state, then repeatedly invokes it
for(output in test_state)
    print(output);
for(output in test_state)
    print(output);
```

## Bindings

Gammakit provides bindings. These are basically the standard library. Bindings act just like global user-defined functions, except instead of moving interpreter control over to a new chunk of code an opening a new frame, they run some native code mid-interpreter-execution, which is pretty important if you want to do things like `print()`. Various aspects of gammakit as already described would make no sense or be downright impossible without bindings.

### Arrow bindings

Arrow bindings are special bindings that look like methods and are invoked on arbitrary variables (or expressions) with syntax like `var->arrow_binding();`. Arrow bindings are for things like "length of this array", "mutably pop the value from the back of this array and return it", etc. This syntax (rather than using .) solely comes from the fact that gammakit is dynamically typed and can't know whether the expression to the left of the -> (or .) is an instance identifier, a number, a string, or what. Here's why it needs to know: `myvariable.someidentifier()` would be ambiguous about whether `someidentifier` is a special binding of some sort or a method in some instance, because the type of `myvariable` is not known, so the interpreter would have to access `myvariable` and check its type at runtime to figure out what `someidentifier` even is. This is quite nasty, so I decided to use a separate syntax.

```gml
var xaa = "myarray";
print(xaa);
xaa->replace_char(1, "Ｙ");
print(xaa);
```

One of the points of arrow bindings is that they do not copy the value out from inside of the variable they're called on if called on a variable (they work on expressions too); this means that it is not useful to be able to declare new arrow bindings from inside of gammakit code, because gammakit does not have variable-level reference semantics, and arrow bindings declared inside of gammakit code would not be able to give the same guarantee. This is important when, for example, calling an arrow bindings on a massive array. As such, it is not possible to declare arrow bindings from inside gammakit code. You should use an instance and a method on it instead.

## Embedding

The primary usage scenario for gammakit is being embedded inside of game engines. As a result, gammakit's built-in suite of bindings is quite limited. The only IO mechanism provided is printing to the console.

Gammakit is designed as a library where the parent application can create and run any number of interpreters in any way that it wants, even going as far as being able to step them instruction-by-instruction. It's also possible to insert your own bindings (obviously) as lambdas that carry in, for example, mutable smart pointers or mutexes, so that bindings can Do Things. This is exactly what the toy game engine "magmakit" does, providing things like a window, keyboard/mouse input, and basic 2d rendering.

Gammakit suspiciously lacks a module system. Instead of having modules and having them jump from one to the other, gammakit's interpreter can have new code loaded into it arbitrarily. This is how magmakit's framerate limiter and input updates work. Rather than being called on in gammakit itself, they silently happen between drawing and stepping, in a loop.

But the embedding application can go about this in any way that it wants, such as having a binding that cycles the rendering system, a binding that cycles the framerate limiter, and a binding that cycles OS input updates, calling them at the tail end of a single massive while() loop. The point being that the embedding application can do whatever it wants; different approaches make sense for different applications, either for design reasons or because of library/OS constraints.

The parent application also has the ability to construct arbitrary "custom" values consisting of two u64s. The first u64 is assumed to be an arbitrary "type" indicator, and the second is assumed to be an arbitrary "opaque pointer" to some information of the type indicated by the first u64. Gammakit might make it possible to compare whether two "custom" values are of the same "type" in the future, and it's already possible to test whether they're exactly the same, but the "custom" type is not compatible with ordered inequality operations, or indeed any operations other than == and != in general.

The embedding application can, also, decide to not insert the interpreter's default bindings, just like with lua.
