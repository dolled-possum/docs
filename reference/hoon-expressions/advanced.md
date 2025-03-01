+++
title = "Advanced Types"
weight = 5
template = "doc.html"
+++
The definition of `type` in the 'Basic Types' section is only a simplified version.  The Hoon type system is simple, but not **that** simple.

The good news is that you can **skip this section**, at least when
you're first learning Hoon.  Polymorphism and aliasing are mainly
for advanced programmers writing complex infrastructure and/or
large functions.  Don't worry about them for a little while.

## The Full Version of `$type`

```
+$  type  $~  %noun
          $@  $?  %noun
                  %void
              ==
          $%  [$atom p=term q=(unit @)]
              [$cell p=type q=type]
              [$core p=type q=coil]
              [$face p=$@(term tune) q=type]
              [$fork p=(set type)]
              [$hint p=(pair type note) q=type]
              [$hold p=type q=hoon]
          ==
::
+$  coil  $:  p=[p=(unit term) q=poly r=vair]
              q=type
              r=[p=seminoun q=(map term tome)]
::
+$  poly  ?(%wet %dry)
+$  vair  ?($gold $iron $lead $zinc)
+$  tome  (pair what (map term hoon))
+$  tune
          $~  [~ ~]
          $:  p=(map term (unit hoon))
              q=(list hoon)
          ==
```

If you compare this to the basic `type`, you'll see that we've
changed two parts: `%core` and `%face`.  We added polymorphism to
`%core`, and aliases and bridges to `%face`.

(We also added `%hint` for hint expressions.)

## `%core`: advanced polymorphism

If cores never changed, we wouldn't need polymorphism.  Of
course, nouns are immutable and never change, but we use them as
templates to construct new nouns around.

Suppose we take a core, a cell `[battery payload]`, and replace
the payload with a different noun.  Then, we invoke an arm from
the battery.

Is this legal?  Does it make sense?  Every function call in Hoon
does this, so we'd better make it work well.

The full core stores both payload types.  The type that describes
the payload currently in the core is `p`.  The type that describes
the payload the core was compiled with is `q.q`.

In the Bertrand Meyer tradition of type theory, there are two
forms of polymorphism: variance and genericity.  In Hoon this
choice is per core, hence the `poly` label in `coil`: it can be either `%wet` or `%dry`.  Dry polymorphism relies on variance; wet
polymorphism relies on genericity.

### Dry arms

For a dry arm, we apply the Liskov substitution principle: we
ask, "can we use any `p` as if it was a `q.q`?"  This is the same
test as in `^-` ("kethep") or any type comparison (`nest`).  Intuitively,
we ask: "is the new payload compatible with the old payload?"

For a core `a`, if `p.a` fits in `q.q.a`, we can use arms on the
core.  The product of a dry arm whose expression is `b` is always
defined as `[%hold a(p q.q.a, r.p.q %gold) b]`.

In other words, the subject of the arm computation is the
original core type.  The type of the modified payload, `p`, is
thrown away.  This is of course the normal behavior of a function
call in most languages.

### Dry polymorphism and core nesting rules

Dry polymorphism works by substituting cores.  Typically, the
the programmer uses one core as an interface definition, then
replaces it with another core which does something useful.

For core `b` to nest within core `a`, the batteries of `a` and
`b` must have the same tree shape, and the product of each `b`
arm must nest within the product of the `a` arm.  Wet arms (see
below) are not compatible unless the hoon is exactly the same.

But we also apply a payload test that depends on the rules of
variance.  Again, this is traditional Meyer type theory; don't
worry if you don't know it.  These rules should make sense if you
just think about them intuitively.

Each core has a "metal" `r.p.q` which defines its variance model.
A core can be **invariant** (`%gold`), **bivariant** (`%lead`),
**covariant** (`%zinc`), or **contravariant** (`%iron`).  The default
is gold; a gold core can be cast or converted to any metal, and
any metal can be cast or converted to lead.

A gold core `a` has a read-write payload; another core `b` that
nests within it (i.e., can be substituted for it) must be a gold
core whose payload is mutually compatible (`+3.a` nests in `+3.b`,
`+3.b` nests in `+3.a`).  Hence, **invariant**.

A lead core `a` has an opaque payload.  There is no constraint on
the payload of a core `b` which nests within it.  Hence,
**bivariant**.

(_Opaque_ here means that the faces and arms are not exported into
the namespace, and that the values of faces and arms can't be written to.
The object in question can be replaced by something else without breaking type
safety.)

An iron core `a` has a write-only sample (payload head, `+6.a`)
and an opaque context (payload tail, `+7.a`).  A core `b` which
nests within it must be a gold or iron core, such that `+6.a`
nests within `+6.b`.  Hence, **contravariant**.

A zinc core `a` has a read-only sample (payload head, `+6.a`)
and an opaque context (payload tail, `+7.a`).  A core `b` which
nests within it must be a gold or zinc core, such that `+6.b`
nests within `+6.a`.  Hence, **covariant**.

### Wet arms

For a wet arm, we ask: "suppose this core was actually compiled
using `p` instead of `q.q`?"  Would the Nock formula we generated
for `q.q` actually work for a `p` payload?

A wet arm with hoon `b` in core `a` produces the type `[%hold a
b]`.  Assuming `a` has a modified payload, this will (lazily)
analyze the arm with the caller's modified payload, `p`.

But of course, we don't actually recompile the arm at runtime.
We actually run the formula generated for the original payload,
`q.q`.

When we call a wet arm, we're essentially using the hoon as a
macro.  We are not generating new code for every call site; we
are creating a new type analysis path, which works as if we
expanded the callee with the caller's context.

Consider a function like `turn` (Haskell `map`) which transforms
each element of a list.  To use `turn`, we install a list and a
transformation function in a generic core.  The type of the list
we produce depends on the type of the list and the type of the
transformation function.  But the Nock formulas for transforming
each element of the list will work on any function and any list,
so long as the function's argument is the list item.

Again, will this work?  A simple (and not quite right) way to ask
this question is to compile all the hoons in the battery for both
a payload of `p` and a payload of `q.q`, and see if they generate
exactly the same Nock.  The actual algorithm is a little more
interesting, but not much.

A Haskeller might say that in a sense, `q.q` and `q.r.q` (the
original payload and the battery) define a sort of implicit
typeclass.  And indeed, Hoon uses wet arms for the same kinds of
problems as Haskell typeclasses.

Like typeclasses, wet arms / gates are a powerful tool.  Don't
use them unless you know what you're doing.

### Constant folding

There's only one field of the `coil` we haven't explained yet:
`p.r.q`.  This is simply the compiled battery, if available.  (Of
course, we compile the hoons in a core against the core itself,
and the formulas can't be available while we're compiling them.)
External users of the core want this battery constant, though: it
lets us fold constants by executing arms at compile time.

## `%face`: aliases and bridges

In the advanced `tune` form, the `%face` type also has a couple
of secret superpowers for hacking the namespace.  Remember that
Hoon doesn't have anything like a symbol table; to resolve a
limb, we just search the type depth-first.

If a name is in the `p.p` map, it's an alias.  (An alias is defined using the
`=*` rune.) The map contains a `(unit hoon)`; if the unit is full, the name
resolves to that hoon (compiled against the `q` type).  If the unit is empty,
the name is blocked / skipped (see [limb](./docs/reference/hoon-expressions/limb/limb.md) for what
this means).

If a name is in the `q.p` map, it's a bridge.  (A bridge is defined using the
`=,` rune.)  When we search for a name, we also compile the bridge, and check
if the name resolves against the bridge product.  If so, we use it.

These hacks let us deal with interesting large-scale problems,
like complex library dependencies.  You definitely don't need to
understand them to just write simple Hoon programs.
