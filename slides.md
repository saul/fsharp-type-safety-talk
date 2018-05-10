class: bold

Stronger Type Safety in F#
--------------------------

### Saul Rennison (RQF)

---

name: agenda

# Agenda

1. Introduction
1. Type safety in F#
1. Generalised ADTs
1. Existential types
1. Annulus of power
1. References

???

- don't sound like a zealot!
- the things f# gives us to help with type saftey
    - no nulls, options
    - single-case DUs / tiny-types
    - DU - can be done in OO but here is v easy (first class visitor pattern)
- the things we've built on this
    - GADTs - see Remco's noddy example - parser
    - existentials - from first principles
    - HLists
    - 'anulus of power'
        - input: unsafe - command args &c
        - inner: super-safe bit
        - need power to bridge this gap - existentials &c

---

template: agenda
class: agenda, agenda-1

---

# Hello!

☑️ Hi, I'm Saul

☑️ Joined G-Research as a grad in Sept 2016

☑️ Previously an intern in RQRD (now RQS & RQF) for 3 months in 2015

--

.right-image[
![](./youre-wrong.jpg)
]

❌ An expert in functional programming

❌ Here to be a zealot

❌ Speaking from an ivory tower

---

# Introduction

Whirlwind tour of how F# enables stronger type safety compared to other languages.

Some of the concepts have been talked about more thoroughly already.

Others have only been touched upon and I'll expand on.

.important[
    Key takeaways will be starred and highlighted like this.
]

**Let's go!**

---

template: agenda
class: agenda, agenda-2

---

class: bold

# `NullReferenceException: Object reference not set to an instance of an object`

---

class: middle, center

![](./sad-panda.jpg)

---

# 

.right-image[
![](./tony-hoare.jpg)
]

> I call it my billion-dollar mistake. It was the invention of the null reference in 1965.

--

> My goal was to ensure that **all use of references should be absolutely safe**, with checking performed automatically by the compiler.

--

> But I couldn't resist the temptation to put in a null reference, simply because it was so easy to implement.
>
> -- <cite>Tony Hoare<cite>

---

# What's wrong with `null`?

`null` is a value used to represent the lack of a value. This is the problem.

--

1. subverts types .muted[(can be used for any reference type)]
1. is a special case .muted[(has to checked for everywhere)]
1. makes poor APIs .muted[(e.g. `String.IsNullOrEmpty`)]
1. difficult to debug .muted[(how did this become null?)]
1. plus many, many more

--

When your type system (e.g. Java, or C#) allows `null` everywhere, you cannot exclude the possibility of `null`.

It's inevitable it will wind up conflated somewhere.

--

**We can do better!** 💪

---

name: nulls-in-fsharp

# Nulls in F#&nbsp;

Definition of a simple *record* type in F#:

```f#
type MyLittleType = {
    Foo : string
    Bar : int
}
```

---

template: nulls-in-fsharp

The above is similar to this C# (and essentially compiles down to):

```c#
public class MyLittleType { // Equatable, Comparable, etc.
    public MyLittleType(string foo, int bar) {
        this.Foo = foo;
        this.Bar = bar;
    }

    public readonly string Foo { get; }
    public readonly int Bar { get; }
}
```

---

template: nulls-in-fsharp

<hr>

```f#
let baz = { Foo = "Hello world"; Bar = 5 }
```

.success[OK!]

--

<hr>

```f#
let baz : MyLittleType = null
```

--

.error[
    `error FS0043: The type 'MyLittleType' does not have 'null' as a proper value`
]

---

class: bold

# So how do you represent `null`?

---

# Why do we need `null`?

`null` in OO languages represents the lack of an instance.

**So why don't we encode that in the type system?**

---

# Discriminated union types

aka _tagged unions_ / _sum_ types.

```f#
type Option<'T> =
    | None
    | Some of value: 'T
```

`Option<'T>` is part of the F# core library and represents a missing value, or a value.

.important[
Seeing `null` anywhere in F# is a huge code smell. 💩
]

---

# Discriminated union types

aka _tagged unions_ / _sum_ types.

```f#
type Option<'T> =
    | None                  // parameterless case
    | Some of value: 'T     // case with a single generic parameter
```

---

# Using discriminated unions

```f#
let nothing = None  // This can satisfy any 'T in Option<'T>
val nothing : Option<'T>

let instance = Some { Foo = "Hello world"; Bar = 5 }
val instance : Option<MyLittleType>
```
--
<hr>

Pulling values out of cases:

```f#
match instance with
| None ->
    printfn "There is nothing here 😢"
| Some underlyingValue -> // Pattern is `Some underlyingValue`
    printfn "Foo is '%s'" underlyingValue.Foo
    // ==> Foo is 'Hello world'
```

This is called _pattern matching_.

Can also pattern match on strings, integers, records, lists, arrays - very powerful.

---

# Exhaustive pattern matching

```f#
let checkForEmpty (foo : Option<MyLittleType>) =
    match foo with
    | Some underlyingValue ->
        printfn "Bar is '%d'" underlyingValue.Bar
```

--

.error[
    `error FS0025: Incomplete pattern matches on this expression. For example, the value 'Some (_)' may indicate a case not covered by the pattern(s).`
]

.important[
Every case must be handled.
]

---

class: bold

# We've hit Tony Hoare's goal! 🎉

> My goal was to ensure that all use of references should be absolutely safe, with **checking performed automatically by the compiler**.

---

# Aside

Cheekily, `None` compiles down to `null` in the IL.

This is a performance optimisation under-the-hood.

This is fine because the **dev never has to care about nulls** - the compiler takes care of that for us.

---

# What `Option` compiles to

![](option.png)

---

# Discriminated unions in C#&nbsp;

TODO: It's possible to model F# DU types in C# as a _visitor pattern_.

---

# Coming to C# 8.x

C# is moving towards 'non-nullable by default' in C# 8.0:

```c#
class Person
{
    public string First;   // Not null
    public string? Middle; // May be null
    public string Last;    // Not null
}
```

---

# Coming to C# 8.x

Also implementing records:

```c#
class Person(string First, string Last);
```

Translates to

```c#
class Person : IEquatable<Person>
{
    public string First { get; }
    public string Last { get; }

    public Person(string First, string Last);

    public void Deconstruct(out string First, out string Last);

    public bool Equals(Person other);

    public override bool Equals(object obj);
    public override int GetHashCode();
}
```

---

# Coming to C# 8.x

Also implementing pattern matching _expressions_:

```c#
class Person(string First, string Last);

var someResult =
    person switch
    {
        Professor p => $"Prof. {p.First} {P.Last}";
        Student p => $"{p.First} {P.Last} (Student)";
        _ => $"Unknown";
    };
```

---

# (Mis-)using built-in types

Built-in types, like ints and strings, are often used to represent values that must fit some constraints.

These constraints are not apparent from the type system.

--

For example:

```c#
public bool SendEmail(string from, string to, string body)
```

`string` can represent an arbitrary sized sequence of UTF-16 code points.

**There are definitely some strings which are not valid email addresses.**

The example above is _stringly typed_.

---

# Single-case discriminated unions

A solution to the _stringly typed_ problem is to use _tiny types_ (aka single-case discriminated unions).

```f#
type EmailAddress = EmailAddress of string
```

This is another instance of a union type introduced earlier, but with just a **single case**.

--

```f#
module EmailAddress =
    let tryParse (email : string) =
        // Yes, this is quite weak!
        if Regex.IsMatch(email, @"[a-z]+@[a-z]+.[a-z]+") then
            Some (EmailAddress email)
        else
            None
```

---

# Single-case discriminated unions

```f#
type EmailAddress = EmailAddress of string
```

<hr>

```f#
let myEmail = EmailAddress.tryParse "saul.rennison@gresearch.co.uk"

match myEmail with
| None -> printfn "Could not parse"
| Some email ->
    let emailParts = email.Split '@'
    // ...
```

--

.error[
    `error FS0039: The field, constructor or member 'Split' is not defined.`
]

We can't implicitly access the underlying string.

The only methods/functions available for `EmailAddress` are ones we write.

---

# Single-case discriminated unions

```f#
// In EmailAddress.fsi
type EmailAddress
```

<hr>

Hiding the implementation of the type forces you to put your domain logic in the same place:

```f#
type EmailAddress = EmailAddress of string

module EmailAddress =
    let tryParse (email : string) = // ...
    let localPart (EmailAddress email) =
        email.Split('@').[0]
```

.important[
The only way to construct or act on an `EmailAddress` is via the functions on its module.

It is an **opaque type**.
]

---

# Single-case discriminated unions

```f#
// In EmailAddress.fsi
type EmailAddress
```

<hr>

In some other file:

```f#
let isGmailAddress (email : EmailAddress) =
    match email with
    | EmailAddress underlyingString -> // error FS0039
        underlyingString.EndsWith "@gmail.com"
```

.error[
`error FS0039: The pattern discriminator 'EmailAddress' is not defined.`
]

--

TomD did a great talk on domain modelling last Thursday.

---

# Single-case discriminated unions

```c#
public bool SendEmail(string from, string to, string body)
```

Can now be re-written as:

```f#
let sendEmail
    (from: EmailAddress)
    (to: EmailAddress)
    (body: string)
    : bool
```

This is more **strongly typed**.

It is impossible for us to pass an invalid email address to `sendEmail`.

---

# Single-case discriminated unions

Key takeaways:

☑️ Use the type system to represent different types of values

☑️ No implicit conversions

☑️ Opaque types allow us to hide the underlying value (e.g a string)

---

template: agenda
class: agenda, agenda-3

---

class: bold

# Why GADTs?

## We want to make invalid expressions _compile-time failures_.

Typically used to represent domain-specific languages (DSLs).

---

# Simple DSL

Imagine a basic arithmetic language which can represent:

- Addition
- Integer constants
- Comparing to zero
- If expressions

---

# Basic Example

```f#
type Expr =
    | Const of int
    | Add of Expr * Expr
    | IsZero of Expr
    | If of Expr * Expr * Expr
```

--

☑️ Allows us to express:

```f#
let valid = Add (Const 1) (Const 1)
```

--

❌ Also allows:

```f#
let invalid1 = Add (IsZero (Const 1)) (Const 1)
let invalid2 = If (Const 1) (Const 2) (Const 3)
let invalid3 = If (Const 1) (IsZero (Const 2)) (Const 3)
```

These don't make any sense! 👎

---

# What we're missing

We couldn't differentiate between expressions of different types.

--

Can generics help us?

```f#
type Expr<'T> =
    | Const of int
    | Add of Expr<int> * Expr<int>
    | IsZero of Expr<int>
    | If of Expr<bool> * Expr<'T> * Expr<'T>
```

--

❌ But we can still do:

```f#
let bad : Expr<bool> = Const 1
```

`Const 1` is an integer expression, not a `bool`! 👎

---

# The problem with generics

Each discriminated union constructor is essentially a function:

```f#
val Const : int -> Expr<'T>
val Add : Expr<int> -> Expr<int> -> Expr<'T>
val IsZero : Expr<int> -> Expr<'T>
val If : Expr<bool> -> Expr<'T> -> Expr<'T> -> Expr<'T>
```

Only the `If` constructor has the return type we want.

---

# In an ideal world

```f#
type Expr<'T> =
    | Const of int when 'T: int
    | Add of Expr<int> * Expr<int> when 'T: int
    | IsZero of Expr<int> when 'T: bool
    | If of Expr<bool> * Expr<'T> * Expr<'T>
```

--

.important[
We can't represent this in .NET
]

---

class: center, middle

![](./sad-panda.jpg)

---

# The workaround

In the ideal world, we used equations to constrain `'T` to a particular type.

To encode this in F#, we need a _type equality_ (called a _Teq_ for short).

--

<hr>

`Teq<'T,'U>` has the following properties:

- Can only construct `Teq<'T,'U>` if `'T` and `'U` are the same type

- When we have a `Teq<'T,'U>`, we can treat `'T` as `'U` and vice versa.

Acts as evidence that `'T` is equal to `int` (for example).

---

# Teq module

```f#
type Teq<'T, 'U> // Opaque type

module Teq =
    val refl<'T> : Teq<'T,'T>
    val cast<'T,'U> : Teq<'T,'U> -> 'T -> 'U
```

We have used an **opaque type** again to expose only our constructor.

We can only instantiate a `Teq<'T, 'U>` through `Teq.refl`.

The type signature of `Teq.refl` ensures that `'T` is the same as `'U`

--

<hr>

`Teq.cast` allows us to cast a `'T` to a `'U`.

This allows us to treat `'T` as `'U` and vice versa.

---

class: middle, center

![](what.jpg)

---

# Using Teqs to achieve type safety

```f#
type Expr<'T> =
    | Const of Teq<int, 'T> * int
    | Add of Teq<int, 'T> * Expr<int> * Expr<int>
    | IsZero of Teq<bool, 'T> * Expr<int>
    | If of Expr<bool> * Expr<'T> * Expr<'T>
```

--

Let's try our previous example again:

```f#
let bad : Expr<bool> = Const (Teq.refl, 1)
```

--

`Teq.refl` would have to return `Teq<int, bool>`. **This won't compile!** 🎉

---

# Using Teqs

Let's write an interpreter for `Expr`:

```f#
let rec eval<'T> (e: Expr<'T>) : 'T =
    match e with
    | Const (teq, value) ->
        value // Error!
```

The error here is because `value` is `int`, but our return type is `'T`.

How can we tell the compiler that we _know_ `'T` and `int` are the same type?

--

<hr>

Remember our Teq! We have `teq` of type `Teq<int, 'T>`.

We can use `Teq.cast teq value` to cast `value` to `'T`.

The compiler is now 😃. Completely type safe!

---

# The whole hog

```f#
let rec eval<'T> (e: Expr<'T>) : 'T =
    match e with
    | Const (teq, value) ->
        Teq.cast teq value

    | Add (teq, left, right) ->
        let result = eval left + eval right
        Teq.cast teq result

    | IsZero (teq, expr) ->
        let result = eval expr = 0
        Teq.cast teq result

    | If (condition, thenExpr, elseExpr) ->
        if eval condition then 
            eval thenExpr
        else
            eval elseExpr
```

---

# Implementing Teq

So I haven't mentioned how this whimsical `Teq` is implemented.

It's a lot simpler than you'd imagine!

--

TODO: why we have 'a->'b

```f#
type Teq<'T, 'U> = Refl of ('T -> 'U) * ('U -> 'T)
```

--

```
module Teq =
    let refl<'T> : Teq<'T,'T> = Refl (id, id)

    let cast<'T,'U>
        (Refl (tToU, uToT) : Teq<'T, 'U>)
        (value : 'T)
        : 'U =
        tToU value
```

---

# GADTs

.important[
This isn't just an academic exercise - we use Teq often.

See Simon's talk tomorrow for real world examples.
]

Key takeaways:

☑️ 

☑️ 

☑️ 

---

template: agenda
class: agenda, agenda-4

---

template: agenda
class: agenda, agenda-6

---

# References

- **The future of C# | Microsoft Build 2018**<br>
    https://channel9.msdn.com/Events/Build/2018/BRK2155

- **DogeConf 2018: Domain Modelling** - _Tom Davies_<br>
    Video should be available soon

- **Type-safe domain-specific lanagues in F#** - _Remco Bras_<br>
    Coming to the GR blog soon