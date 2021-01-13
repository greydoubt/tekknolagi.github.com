---
title: Inline caching
layout: post
date: 2020-01-13 11:00:00 PT
---

Inline caching is a popular technique for runtime optimization. It was first
introduced in 1984 in Deutsch &amp; Schiffman's paper [Efficient implementation
of the smalltalk-80 system][smalltalk] but has had a long-lasting legacy in
today's dynamic language implementations. Runtimes like the Hotspot JVM, V8,
and SpiderMonkey use it to improve the performance of code written for those
virtual machines.

[smalltalk]: http://web.cs.ucla.edu/~palsberg/course/cs232/papers/DeutschSchiffman-popl84.pdf

In this blog post, I will attempt to distill the essence of inline caching
using a small and relatively useless bytecode interpreter built solely for this
blog post.

## Background

In many compiled programming languages like C and C++, types and attribute
locations are known at compile time. This makes code like the following fast:

```c
#include "foo.h"

Foo do_add(Foo left, Foo right) {
  return left.add(right);
}
```

The compiler knows precisely what type `left` and `right` are (it's `Foo`) and
also where the method `add` is in the executable. If the implementation is in
the header file, it may even be inlined and `do_add` may be optimized to a
single instruction. Check out the assembly I got with `objdump`:

```
0000000000401160 <_Z6do_add3FooS_>:
  401160:	48 83 ec 18          	sub    $0x18,%rsp
  401164:	89 7c 24 0c          	mov    %edi,0xc(%rsp)
  401168:	48 8d 7c 24 0c       	lea    0xc(%rsp),%rdi
  40116d:	e8 0e 00 00 00       	callq  401180 <_ZN3Foo3addES_>
  401172:	48 83 c4 18          	add    $0x18,%rsp
  401176:	c3                   	retq   
```

All it does is save the parameters to the stack, call `Foo::add`, and then
restore the stack.

In more dynamic programming languages, it is often impossible to determine at
runtime startup what type any given variable binding has. I'll use Python as an
example to illustrate how dynamism makes this tricky, but this constraint is
broadly applicable to Ruby, JavaScript, etc.

Consider the following Python snippet:

```python
def do_add(left, right):
    return left.add(right)
```

Due to Python's various dynamic features, the compiler cannot in general know
what type `value` is and therefore what code to run when reading `left.add`.
This program will be compiled down to a couple Python bytecode instructions
that do a very generic `LOAD_METHOD`/`CALL_METHOD` operation:

```
>>> import dis
>>> dis.dis("""
... def do_add(left, right):
...     return left.add(right)
... """)
[snip]
Disassembly of <code object do_add at 0x7f0b40cf49d0, file "<dis>", line 2>:
  3           0 LOAD_FAST                0 (left)
              2 LOAD_METHOD              0 (add)
              4 LOAD_FAST                1 (right)
              6 CALL_METHOD              1
              8 RETURN_VALUE

>>> 
```

This `LOAD_METHOD` Python bytecode instruction is unlike the x86 `mov`
instruction in that `LOAD_METHOD` is not given an offset into `left`, but
instead is given the name `"add"`. It has to go and figure out how to read
`add` from `left`'s type --- which could change from call to call.

In fact, even if the parameters were typed (which is a new feature in Python
3), the same code would be generated. Writing `left: Foo` means that `left` is a
`Foo` *or* a subclass.

This is not a simple process like "fetch the attribute at the given offset
specified by the type". The runtime has to find out what kind of object `add`
is. Maybe it's just a function, or maybe it's a `property`, or maybe it's some
custom descriptor protocol thing. There's no way to just turn this into a
`mov`!

... or is there?

## Runtime type information

Though dynamic runtimes do not know ahead of time what types variables have at
any given opcode, they do eventually find out *when the code is run*. The first
time someone calls `do_add`, `LOAD_METHOD` will go and look up the type of
`left`. It will use it to look up the attribute `add` and then throw the type
information away. But the second time someone calls `do_add`, the same thing
will happen. Why don't runtimes store this information about the type and the
method and save the lookup work?

The thinking is "well, `left` could be any type of object --- best not make any
assumptions about it." While this is *technically* true, Deutsch &amp;
Schiffman find that "at a given point in code, the receiver is often the same
class as the receiver at the same point when the code was last executed".

> **Note:** By *receiver*, they mean the thing from which the attribute is
> being loaded. This is some Object-Oriented Programming terminology.

This is huge. This means that, even in this sea of dynamic behavior, humans
actually are not all that creative and tend to write functions that see only a
handful of types at a given location.

The Smalltalk-80 paper describes a runtime that takes advantage of this by
adding "inline caches" to functions. These inline caches keep track of variable
types seen at each point in the code, so that the runtime can make optimization
decisions with that information.

Let's take a look at how this could work in practice.

## A small example

I put together a [small stack machine][repo] with only a few operations. There
are very minimal features to avoid distracting from the main focus: inline
caching. Extending this example would be an excellent exercise.

[repo]: https://github.com/tekknolagi/icdemo

### Objects and types

The design of this runtime involves two types of objects (`int`s and `str`s). I
implement this as a tagged union, but for the purposes of this blog post the
representation does not matter very much.

```c
typedef enum {
  kInt,
  kStr,
} ObjectType;

typedef struct {
  ObjectType type;
  union {
    const char *str_value;
    int int_value;
  };
} Object;
```

These types have methods on them, such as `add` and `print`. I represent this
relationship with two levels of tables: a table mapping types to method lists
(`kTypes`), and a sentinel-terminated table mapping method names to function
pointers (`kIntMethods` or `kStrMethods`).

```c
typedef enum {
  kAdd,
  kPrint,

  kUnknownSymbol = kPrint + 1,
} Symbol;

// Note: this takes advantage of the fact that in C, not putting anything
// between the parentheses means that this function can take any number of
// arguments.
typedef Object (*Method)();

typedef struct {
  Symbol name;
  Method method;
} MethodDefinition;

static const MethodDefinition kIntMethods[] = {
    {kAdd, int_add},
    {kPrint, int_print},
    {kUnknownSymbol, NULL},
};

static const MethodDefinition kStrMethods[] = {
    {kAdd, str_add},
    {kPrint, str_print},
    {kUnknownSymbol, NULL},
};

static const MethodDefinition *kTypes[] = {
    [kInt] = kIntMethods,
    [kStr] = kStrMethods,
};
```

While I represent names with an enum (`Symbol`) here, strings would work just
as well.

### Interpreter

There's no way to call these methods directly, though. For the purposes of this
demo, the only way to call these methods is through purpose-built opcodes. For
example, the opcode `ADD` takes two arguments. It looks up `kAdd` on the left
hand side and calls it. `PRINT` is similar.

There are only two other opcodes, `ARG` and `HALT`.

```c
typedef enum {
  // Load a value from the arguments array at index `arg'.
  ARG,
  // Add stack[-2] + stack[-1].
  ADD,
  // Pop the top of the stack and print it.
  PRINT,
  // Halt the machine.
  HALT,
} Opcode;
```

Bytecode is represented by a series of opcode/argument pairs, each taking up
one byte. Only `ARG` needs an argument; the other instructions ignore theirs.

You may wonder, "how is it that there is an instruction for loading arguments
but no call instruction?" Well, the interpreter does not support calls. There
is only a function, `eval_code`. It takes an object, evaluates its bytecode
with the given arguments, and returns.

The implementation is a fairly straightforward `switch` statement. Notice that
it takes a representation of a function-like thing (`Code`) and an array of
arguments. `nargs` is only used for bounds checking.

```c
static unsigned kBytecodeSize = 2;

void eval_code_uncached(Code *code, Object *args, int nargs) {
  int pc = 0;
#define STACK_SIZE 100
  Object stack_array[STACK_SIZE];
  Object *stack = stack_array;
#define PUSH(x) *stack++ = (x)
#define POP() *--stack
  while (true) {
    Opcode op = code->bytecode[pc];
    byte arg = code->bytecode[pc + 1];
    switch (op) {
      case ARG:
        CHECK(arg < nargs && "out of bounds arg");
        PUSH(args[arg]);
        break;
      case ADD: {
        Object right = POP();
        Object left = POP();
        Method method = lookup_method(left.type, kAdd);
        Object result = (*method)(left, right);
        PUSH(result);
        break;
      }
      case PRINT: {
        Object obj = POP();
        Method method = lookup_method(obj.type, kPrint);
        (*method)(obj);
        break;
      }
      case HALT:
        return;
      default:
        fprintf(stderr, "unknown opcode %d\n", op);
        abort();
    }
    pc += kBytecodeSize;
  }
}
```