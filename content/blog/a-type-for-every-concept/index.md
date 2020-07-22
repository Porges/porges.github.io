---
title: A Type For Every Concept
draft: true
---

[⚠ *many opinions*]

I’ve joked in the past with colleagues about wanting a language where primitive types can only be used as representations. At least, they thought I was joking. This is the end-game of avoiding ‘[primitive obsession](http://wiki.c2.com/?PrimitiveObsession)’.

> Primitive types are mere representations!

For every concept your code cares about, there should be a type. I’m not just thinking about `User` or `Person` or `Student` or `Cat` or `Vehicle`, I mean the “little types” as well – you shouldn’t be using primitive types to represent anything that’s within the specific domain of your code.

A username is probably not “just a string”, just like XML is not just a string, an email is not just a string, and a phone number is not just a string. (Also, an index is not a count, and vice versa. More on this in another post.)

So... give them their own types!

**Tip: it’s probably cheap!** Typed languages often have zero-cost (or near-zero-cost) abstraction into new types. This includes C#/F# (structs), Haskell (newtypes), C++ (all classes).

Here are some more reasons. The examples are going to be in C# as it’s my mother tongue, but they are mostly applicable everywhere.

#### Avoid nonsense values

This is also known as “make illegal states unrepresentable”.

In most cases, something that is represented by a primitive type won’t have all of the possible values of that type be valid values. For example, a person’s age or height represented by an `int` can’t be a negative number.

#### Avoid meaningless operations

In addition, not all *operations* will be valid. Does it make sense to multiply two ages? Does it make sense to multiply an age and a height? Wrapping the `int` in a new type allows us to prevent invalid values from being used, and prevents us from using nonsensical operations.

#### Enforcing invariants/structure

Avoid placing validation code in multiple locations, and validate in the constructor/parse method of the type. Also, prefer normalization to simple validation.

Create types with a proper constructors (or `TryParse` method) and use them. If it’s something like a credit card or phone number that will keep a string representation internally, then *normalize* the format of the string so that all equivalent inputs have the same representation.

If you validate these structured strings against a regular expression and then continue to pass a raw string around you’re throwing away valuable information.

Furthermore, in a large or ageing codebase you’ll likely end up repeating the same validation in multiple places (and sometimes with incorrect or differing methods). Declaring a type gives you a point at which to localize validation.

Even internally (not at the edges of the system), prefer types to preconditions. Having used [Code Contracts](https://github.com/Microsoft/CodeContracts) in the past, it is much more effective to lift preconditions to types—attempting to flow preconditions through multiple layers of code is tedious. Having a type also means that if your preconditions change you only have a single location to update.

#### Specifying consistent behaviour

If some identifier should be case-insensitive, you can enforce this with its own type:

```csharp
struct Identifier
    : IEquatable<Identifier>
{
    private static StringComparer Comparer
        = StringComparer.OrdinalIgnoreCase;

    public Identifier(string value)
    {
        Value = value;
    }

    public string Value { get; }
    public override string ToString() => Value;

    public override int GetHashCode()
        => Comparer.GetHashCode(Value);

    public bool Equals(Identifier other)
        => Comparer.Equals(Value, other.Value);

    public override bool Equals(object obj)
    {
        var other = obj as Identifier?;
        return other.HasValue && Equals(other.Value);
    }

    public static bool operator ==(Identifier left, Identifier right)
        => left.Equals(right);

    public static bool operator !=(Identifier left, Identifier right)
        => !left.Equals(right);
}
```

Unfortunately, this is a little verbose in C#. However, you don’t need to think about it that much as it’s all boilerplate.

In fact, if you ever see a `Dictionary<string, T>` being created with `StringComparison.CaseInsensitive`, it’s a good sign that the `string` should be changed into a distinct type. This ensures that the treatment of the value is the same everywhere that it’s used.

Not only should you use types to enforce particular behaviour, you can use them to *remove* behaviour. If a password should never be printed, you can override `ToString` to print `"⛔"`. If you calculate a floating-point value that shouldn’t be compared for equality, throw `NotSupportedException` in `Equals` (unfortunately you can’t remove equality completely in C#).

#### Easing changes of representation

I recently worked on a project where I introduced a type `Message` as a wrapper around a `string`. I thought this might have been going too far, but I did it anyway. <small><span style="color: lightgrey">yolo</span></small>

Later on, I decided that the message never needed to be deserialized from bytes to a string in the first place, as the code never attempted an interpretation of the raw byte values. I changed the internal representation of `Message` to be a `byte[]` array. This change required only *2 lines* to be modified across the codebase, despite `Message` being stored and mentioned in many places. If I had simply used `string` this would have been much more disruptive.

#### Type safety

This is low down the list because it’s nearly trivial. Everyone should want to avoid swapping samely-typed arguments (e.g. `Login(string user, string password)`). Having a type per concept prevents this. The same method with a type per concept (`Login(UserName user, Password password)`) is unconfusable.

This is something people bump into often with .NET’s `ArgumentNullException(string parameterName, string message)` and `ArgumentException(string message, string parameterName)`. If `nameof` returned a type `Identifier` this would be easily preventable.

#### Preventing naming coincidences

This `string gobble` argument is not the same kind of *gobble* as the `string gobble` argument on that other type in a distant part of the codebase. Distinct types prevent programmer confusion.