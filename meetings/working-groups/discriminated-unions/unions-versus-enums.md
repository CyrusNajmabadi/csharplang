# Unions versus Enums

This proposal aims to separate out two related union-esque concepts that have been discussed for quite some time.  Instead of a single language feature that attempts to unify them into one construct, an approach is taken to keep them separated, though with one built on top of the other.

## Starting point

This proposal builds off of the current [Unions](https://github.com/dotnet/csharplang/blob/38fd5f33d285cb190268f98cea16223cc0a5b8bc/proposals/unions.md) proposal, leaving it virtually unchanged.  Importantly, it maintains the view that a `Union` is effectively just a way of specifying the set of `Types` that a value could be.  The exact syntax of to define a `Union` is not important to this proposal.  But, for the purposes of discussion is presumed to be something like:

```c#
union StringOrInt
{
    int,
    string
}
```

It is also presumed that unions are somewhat normal declarations, that themselves could themselves have members:

```c#
union StringOrInt
{
    int,
    string;

    public bool IsValue()
    {
        return this switch { ... };
    }
}
```

And that unions have a way to define their own types, avoiding the need to clutter the outer namespace:

```c#
union StateMachine
{
    StartState,
    InProgress,
    Completed,

    public record StartState(...) { ... }
    public record InProgress(...) { ... }
    public record Completed(...) { ... }
}
```

Importantly, there is no shorthand in a union to both declare a new type, and make that type part of the types of the union itself.

## Future continuation

Unions solve the problem of being able to represent a value with a finite set of types well.  However, they are not ideal when for all use cases.  In some domains it is preferable to instead have a single type, with a set of rich cases (i.e. 'shapes') that an instance of that type could be.  This is commonly known as an 'algebraic data type'.  C# already has a poor-man's form of this with 'enums'.  But our current enums come with several major drawbacks.

1. They cannot represent information more interesting than an integral value.
2. They are not exhaustive on the total set of values you may actually run into at runtime.

In other words, they work well in domains where one wants to model things as a (possibly open ended) set of named integral values.  But are not very useful beyond that.

While 'Unions' has experimented with solutions for this, the problem spaces seem quite different.  One is for bringing a disparate set of types together.  One is for providing a disparate set of shapes for a particular type.

To that end, to solve the latter need, this proposal recomends 