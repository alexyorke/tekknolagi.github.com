---
title: "Compiling typed Python"
layout: post
description: With a little effort, you can make your mypy-typed Python go zoom.
date: 2023-06-19
---

It's been nine whole years since [PEP 484](https://peps.python.org/pep-0484/)
landed and brought us types from on high. This has made a lot of people very
angry and been widely regarded as a bad move[^adams]. Since then, people on the
internet have been clamoring to find out: [does this mean we can now compile
Python](https://discuss.python.org/t/cpython-optimizations-leveraging-type-hints-bachelor-thesis-topic/27264)
[to native code for more speed](https://stackoverflow.com/questions/40204951/python-3-type-hints-for-performance-optimizations)?
It's a totally reasonable question. It was one of my first questions when I
first started working on Python compilers. So can we do it?

[^adams]: According to Douglas Adams, anyway.

No. But also, kind of, yes. I'll explain. I'll explain in the context of
"ahead-of-time" compiling within or adjacent to CPython, the predominant
implementation of the Python language. Just-in-time (JIT) compilers are a
different beast, and are described more below.

The core thesis is: *types are very broad hints and they are sometimes lies*.

## It's not what you think

A lot of people enjoy walking into discussions and saying things like "We have
a 100% Mypy typed codebase. I would simply use the types in compilation to
generate better code." That was the original title of this article, even. "*I
would simply use the types in the compiler.*" But it doesn't really work like
that[^simple-annotations]. I'll show a couple examples that demonstrate why.
The point is not to browbeat you with example after example; the point is to
get people to stop saying "just" and "simply".

[^simple-annotations]: I'm not even going to address fancy type-level
    metaprogramming stuff, because I don't think people ever intended to use
    that for performance anyway. There are [lots of
    uses](https://lukeplant.me.uk/blog/posts/the-different-uses-of-python-type-hints/)
    for Python type annotations. We'll just look at very simple bog-standard
    `x: SomeTypeName` annotations.

For example, look at this lovely little Python function. It's short, typed, and
obviously just adds integers. Easy: compiles right down to two machine
instructions. Right?

```python
def add(x: int, y: int) -> int:
    return x + y

add(3, 4)  # => 7
```

Unfortunately, no. Type annotations of the type `x: T` mean "`x` is an instance
of `T` or an instance of a subclass of `T`."[^exact] This is pretty common in
programming languages, and, thanks to [Barbara
Liskov](https://en.wikipedia.org/wiki/Liskov_substitution_principle), makes
semantic sense most of the time. But it doesn't help performance here. The
dispatch for binary operators in Python is famously [not
simple](https://snarky.ca/unravelling-binary-arithmetic-operations-in-python/)
because of subclasses. This means that there is a lot of code to be executed if
`x` and `y` could be any subclasses of `int`.

[^exact]: The Static Python project sort of has an internal `Exact` type that
    can be used like `x: Exact[int]` to disallow subclasses, but it's not
    exposed. This is, as Carl Meyer explained to me, because Mypy, Pyre, and
    other type checkers don't have a good way to check this type; the Python
    type system does not support this.

```python
class C(int):
    def __add__(self, other):
        return "no"  # sigh
```

It's not obvious how the appropriate `__add__` function is selected in operator
dispatch. **You don't need to understand or really even read the big blob**
that explains it below. You just need to say "ooh" and "aah" and "wow, so many
if-statements."

```python
_MISSING = object()

def sub(lhs: Any, rhs: Any, /) -> Any:
        # lhs.__sub__
        lhs_type = type(lhs)
        try:
            lhs_method = debuiltins._mro_getattr(lhs_type, "__sub__")
        except AttributeError:
            lhs_method = _MISSING

        # lhs.__rsub__ (for knowing if rhs.__rsub__ should be called first)
        try:
            lhs_rmethod = debuiltins._mro_getattr(lhs_type, "__rsub__")
        except AttributeError:
            lhs_rmethod = _MISSING

        # rhs.__rsub__
        rhs_type = type(rhs)
        try:
            rhs_method = debuiltins._mro_getattr(rhs_type, "__rsub__")
        except AttributeError:
            rhs_method = _MISSING

        call_lhs = lhs, lhs_method, rhs
        call_rhs = rhs, rhs_method, lhs

        if (
            rhs_type is not _MISSING  # Do we care?
            and rhs_type is not lhs_type  # Could RHS be a subclass?
            and issubclass(rhs_type, lhs_type)  # RHS is a subclass!
            and lhs_rmethod is not rhs_method  # Is __r*__ actually different?
        ):
            calls = call_rhs, call_lhs
        elif lhs_type is not rhs_type:
            calls = call_lhs, call_rhs
        else:
            calls = (call_lhs,)

        for first_obj, meth, second_obj in calls:
            if meth is _MISSING:
                continue
            value = meth(first_obj, second_obj)
            if value is not NotImplemented:
                return value
        else:
            raise TypeError(
                f"unsupported operand type(s) for -: {lhs_type!r} and {rhs_type!r}"
            )
```

This enormous code snippet is from Brett Cannon's linked post above. It
demonstrates in Python "pseudocode" what happens in C under the hood when doing
`lhs - rhs` in Python.

But let's ignore this problem and pretend that it's not possible to subclass
`int`. Problem solved, right? High performance math?

Unfortunately, no. While we would have fast dispatch on binary operators and
other methods, integers in Python are heap-allocated big integer objects. This
means that every operation on them is a function call to `PyLong_Add` or
similar. While these functions have been optimized for speed over the years,
they are still slower than machine integers. But let's assume that a
sufficiently smart compiler can auto-unbox small-enough big integers into
machine words at the beginning of a function. We can do fast math with those.
If we really want, we can even do fast floating point math, too. Problem
solved? Hopefully?

```python
# You can have fun making enormous numbers! This is part of the Python
# language.
def big_pow(x: int) -> int:
    # Even if you unbox `x` here, the result might still be enormous.
    return 2**x
```

Unfortunately, no. Most math kernels are not just using built-in functions and
operators. They call out to external C libraries like NumPy or SciPy, and those
functions expect heap-allocated `PyLongObject`s. The C-API is simply **not
ready** to expose the underlying functions that operate on machine integers,
and it is also **not ready** for [tagged pointers](/blog/small-objects/). This
would be a huge breaking change in the API and ABI. But okay, let's assume for
the sake of blog post that the compiler team has a magic wand and can do all of
this.

```c
// Store 63-bit integers inside the pointer itself.
static const long kSmallIntTagBits = 1;
static const long kBits = 64 - kSmallIntTagBits;
static const long kMaxValue = (long{1} << (kBits - 1)) - 1;

PyObject* PyLong_FromLong(long x) {
    if (x < kMaxValue) {
        return (PyObject*)((unsigned long)value << kSmallIntTagBits);
    }
    return MakeABigInt(x);
}

PyObject* PyLong_Add(PyObject* left, PyObject* right) {
    if (IsSmallInt(left) && IsSmallInt(right)) {
        long result = Unbox(left) + Unbox(right);
        return PyLong_FromLong(result);
    }
    return BigIntAdd(left, right);
}
```

Then we're set, right?

Unfortunately, there are still some other loose ends to tie up. While you may
have a nice and neatly typed numeric kernel of Python code, it has to interact
with the outside world. And the outside world is often not so well typed. Thank
you to Jeremy Siek and Walid Taha for giving us [gradual
typing][gradual]---this is the reason anything gets typed at all in
Python---but you can't do type-driven compilation of untyped code.

[gradual]: https://wphomes.soic.indiana.edu/jsiek/what-is-gradual-typing/

This means that at the entry to your typed functions, you have to check the
types of the input objects. Maybe you can engineer a system such that a
function can have multiple entry points---one for untyped calls, one for typed
calls, and maybe even one for unboxed calls---but this hypothetical system's
complexity is growing, and fast. And there are a whole host of other
complications and bits of dynamic behavior that I haven't even mentioned.

```python
def typed_function_unboxed(x: int64, y: int64) -> int64:
    return int64_add(x, y)

def typed_function(x: int, y: int) -> int:
    return typed_function_unboxed(unbox(x), unbox(y))

def typed_function_shell(x, y):
    if not isinstance(x, int):
        raise TypeError("...")
    if not isinstance(y, int):
        raise TypeError("...")
    return typed_function(cast(x, int), cast(y, int))

f = make_typed_function(typed_function_shell, typed_function, typed_function_unboxed)
f(3, 4)  # The dispatch gets hairy
```

And it's not just about types, either! For example, variable binding,
especially global variable binding, is a performance impediment. Globals, even
builtins, are almost always overwritable by any random Python code.

"But Max," you say, "Python compiler libraries like Numba clearly work just
fine. Just-in-time compilers have been doing this for years. What's the deal?"

Just-in-time compilers can be so effective because can speculate on things that
are not compile-time constants. Then, if the assumption is no longer true, they
can fall back on a (slower) interpreter. This is how PyPy does its amazing work
specializing data structures; the JIT does not have to handle every case. This
interpreter deoptimization is part of what makes JITs hard to understand and
does not much help with compiling code ahead-of-time.

This is what a PyPy trace might look like, taken from the 2011 paper [Runtime
Feedback in a Meta-Tracing JIT for Efficient Dynamic
Languages](https://dl.acm.org/doi/10.1145/2069172.2069181). Every `guard` is a
potential exit from JITed code to the interpreter:

```python
# inst1.getattr("a")
map1 = inst1.map
guard(map1 == 0xb74af4a8)
index1 = Map.getindex(map1, "a")
guard(index1 != -1)
storage1 = inst1.storage
result1 = storage1[index1]
```

And the other thing is, Numba doesn't exactly compile *Python*...

## Dialects

Numba is great! I cannot emphasize enough how much I am **not** trying to pick
on Numba here. The team put in a lot of hard work and solid engineering and
built a compiler that produces very fast numerical code.

It's possible to do this because Numba compiles a superficially similar
language that is much less dynamic and is focused on numerics. For people who
work with data analytics and machine learning, this is incredible!
Unfortunately, it doesn't generalize to arbitrary Python code.

This is also true of many other type-driven compiler projects that optimize
"Python" code:

* [Cython](https://github.com/cython/cython)/[Pyrex](https://www.csse.canterbury.ac.nz/greg.ewing/python/Pyrex/)
* [MicroPython Viper](https://docs.micropython.org/en/v1.9.3/pyboard/reference/speed_python.html#the-viper-code-emitter)
* [Mojo](https://www.modular.com/mojo)
* [Mypyc](https://github.com/mypyc/mypyc)
* [Shed Skin](https://github.com/shedskin/shedskin)
* [Starkiller](http://michael.salib.com/writings/thesis/thesis.pdf) (PDF)
* [Static Python](https://github.com/facebookincubator/cinder/#static-python)
* [Typed Python](https://github.com/APrioriInvestments/typed_python/blob/dev/docs/introduction.md)

and in particular to optimize numerics:

* [Codon](https://github.com/exaloop/codon)
* [Numba](https://github.com/numba/numba)
* [lpython](https://github.com/lcompilers/lpython)
* [Pyccel](https://github.com/pyccel/pyccel/blob/devel/tutorial/quickstart.md)
* [Pythran](https://pythran.readthedocs.io/en/latest/)
* [Taichi](https://github.com/taichi-dev/taichi)
* [TensorFlow JIT/XLA](https://www.tensorflow.org/xla)
* [TorchScript](https://pytorch.org/docs/stable/jit_language_reference.html)

and probably more that I forgot about or could not find (please feel free to
submit a PR).

It does raises an interesting question, though: what if you intentionally and
explicitly eschew the more dynamic features? Can you get performance in return?
It turns out, yes. Absolutely yes.

## Stronger guarantees

As people have continually rediscovered over the years, Python is hard to
optimize statically. Functions like the following, which "only" do an attribute
load, have no hope whatsoever of being optimized out of context:

```python
def get_an_attr(o):
    return o.value
```

This is because `o` could be any object, types can define custom attribute
resolution by defining a `__getattr__` function, and therefore attribute loads
are equivalent to running opaque blobs of user code.

Even if the function is typed, you still run into the subclass problem
described earlier. You also have issues like instance attributes, deleting
attributes, descriptor shadowing, and so on.

```python
class C:
    def __init__(self):
        self.value = 5


def get_an_attr(o: C):
    return o.value
```

*However*, if you intentionally opt out of a lot of that dynamism, things start
getting interesting. The [Static
Python](https://github.com/facebookincubator/cinder/#static-python) compiler,
for example, is part of the Cinder project, and lets you trade dynamism for
speed. If a module is marked static with `import __static__`, Cinder will swap
out the standard bytecode compiler for the Static Python bytecode compiler,
which *compiles a different language*!

By way of example, the SP compiler will automatically slotify all of the
classes in a SP module. This means that features that used to
work---dynamically creating and deleting attributes, etc---no longer do. But it
also means that attribute loads from static classes are actually only three
machine instructions now. This tradeoff makes the compiler transform *sound*.
Most people like this tradeoff.

```python
import __static__

class C:
    def __init__(self):
        self.value: int = 5

def test(o: C):
    return o.value
# ...
# mov rax,QWORD PTR [rsi+0x10]  # Load the field
# test rax,rax                  # Check if null
# je 0x7f823ba17283             # Maybe raise an exception
# ...
```

Static Python does this just with existing Python annotations. It has some more
constraints than Mypy does (you can't `ignore` your type errors away, for
example), but it does not change the syntax of Python. But it's important to
know that this does not just immediately follow from a type-directed
translation. It requires opting into stricter checking and different behavior
for an entire typed core of a codebase. It requires changing the runtime
representation of objects from header+dictionary to header+array of slots. For
this reason it is (currently) implemented as a custom compiler, custom bytecode
interpreter, and with support in the Cinder JIT. To learn more, check out the
Static Python team's [paper collaboration with
Brown](https://cs.brown.edu/~sk/Publications/Papers/Published/lgmvpk-static-python/),
which explains a bit more about the gradual typing bits and soundness.

```python
# An example snippet that is allowed by Mypy but not by Static Python because
# it would be dangerous.
def foo(x: int) -> int:
    return x

foo("hello")  # type: ignore
```

In this example,

* CPython ignores the annotations completely
* Mypy lets the type mismatch slide due to `type: ignore`
* Static Python does not allow this code to compile

All of these are reasonable behaviors because each project has different goals.

I would be remiss if I did not also mention
[Mypyc](https://github.com/mypyc/mypyc) (the optimizer and code generator for
Mypy). Mypyc is very similar to Static Python in that it takes something that
looks like Python with types and generates type-optimized code. It is different
in that it generates C extensions. Depending on your use case---in particular,
your deployment story---it may be the compiler that you want to use! The Black
formatter, for example, has had [great
success](https://ichard26.github.io/blog/2022/05/compiling-black-with-mypyc-part-1/)
using Mypyc.

Other projects take this further. The [Mojo](https://www.modular.com/mojo)
project, for example, aims to create a much bigger and more visibly different
new language that is a proper superset of Python[^chris]. The Python code it
runs should continue to work as advertised, but modules can opt into less
dynamism by iteratively converting their Python code to something that looks a
little different. They also do a bunch of other neat stuff, but that's outside
the scope of this blog post.

[^chris]: Thank you to [Chris Gregory](https://www.chrisgregory.me/) for
    pushing me to look more at it!

See for example this snippet defining a struct (a new feature) that reads like
a Python class but has some stronger mutability and binding guarantees:

```python
@value
struct MyPair:
    var first: Int
    var second: Int
    def __lt__(self, rhs: MyPair) -> Bool:
        return self.first < rhs.first or
              (self.first == rhs.first and
               self.second < rhs.second)
```

It even looks like the `@value` decorator gives you value semantics.

Per the Mojo [docs](https://docs.modular.com/mojo/programming-manual.html#struct-types),

> Mojo structs are static: they are bound at compile-time (you cannot add
> methods at runtime). Structs allow you to trade flexibility for performance
> while being safe and easy to use.
>
> [...]
>
> In Mojo, the structure and contents of a "struct" are set in advance and
> can't be changed while the program is running. Unlike in Python, where you
> can add, remove, or change attributes of an object on the fly, Mojo doesn't
> allow that for structs. This means you can't use `del` to remove a method or
> change its value in the middle of running the program.

Seems neat. We'll see what it looks like more when it's open sourced.

## Other approaches

[Nuitka](https://github.com/Nuitka/Nuitka) (not mentioned above) is a
whole-program compiler from Python to C. As far as I can tell, it does not use
your type annotations in the compilation process. Instead it uses its own
optimization pipeline, including function inlining, etc, to discover types.
Please correct me if I am wrong!

### In other languages

If you don't like this whole "typed kernel" idea, other compilers like Graal's
[SubstrateVM](https://docs.oracle.com/en/graalvm/enterprise/20/docs/reference-manual/native-image/#native-image)
do some advanced wizardry to analyze your whole program.

SubstrateVM is an ahead-of-time compiler for Java that looks at your entire
codebase as a unit. It does some intense inter-procedural static analysis to
prove things about your code that can help with performance. It also subtly
changes the language, though. In order to do this analysis, it prohibits
arbitrary loading of classes at runtime. It also limits the amount of
reflection to some known feature subset.

## What this means for the average Python programmer

If you are working on a science thing or machine learning project, you most
likely have a bunch of glue around some fast core of hardcore math. If you add
types and use one of the excellent compilers listed above, you will probably
get some performance benefits.

If you are working on some other project, though, you may not have such a clear
performance hotspot. Furthermore, you may not be working with objects that have
fast primitive representations, like `int`. You probably have a bunch of data
structures, etc. In that case, some of the projects above like Mypyc and
Static Python and Nuitka can probably help you. But you will need to add types,
probably fix some type errors, and then change how you build and run your
application.

Unfortunately for everybody who writes Python, a huge portion of code that
needs to be fast is written as C extensions because of historical reasons. This
means that you can optimize the Python portion of your application all you
like, but the C extension will remain an opaque blob (unless your compiler
understands it like Numba understands NumPy, that is). This means that you may
need to eventually re-write your C code as Python to fully reap all the
performance benefits. Or at least [type the
boundary](https://github.com/faster-cpython/ideas/issues/546).

## What this means for other dynamic languages

These questions about and issues with straightforward type-driven compilation
are not unique to Python. The Sorbet team had to do [a lot of
work](https://sorbet.org/blog/2021/07/30/open-sourcing-sorbet-compiler) to make
this possible for Ruby. People ask the same thing about typed JavaScript (like
TypeScript) and probably other languages, too. People will have to work out
similar solutions. People are working on this; there is at least one neat
extant project that I have been asked not to publicize yet.

## Wrapping up

Types are not ironclad guarantees of data layout. Changing the language you are
compiling to prohibit certain kinds of dynamism can help you with performance.
Several projects already do this and it seems to be growing in popularity.

If nothing else, I hope you have more of an understanding of what types mean
from a correctness perspective, from a performance perspective, and how they do
not always overlap.

I also hope you don't come away from this post feeling sad. I actually hope you
feel hopeful! Tons of brilliant engineers are working tirelessly to make your
code run faster.

### Further reading

After publishing this post I came across [Optimizing and Evaluating Transient
Gradual Typing](https://arxiv.org/abs/1902.07808) which adds type checks to
CPython and erases redundant ones, and then realized that this was also cited
in the Brown paper. The whole series of papers is really interesting. And
apparently I was coworkers with Michael Vitousek for over four years. Neat!

<!-- serrano aot js https://dl.acm.org/doi/10.1145/3276945.3276950 -->
<!-- jscomp? https://github.com/tmikov/jscomp -->
<!-- smalls https://github.com/tmikov/smalls -->
<!-- mycpp -->

<br />
<hr style="width: 100px;" />
<!-- Footnotes -->