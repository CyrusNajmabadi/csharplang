# Unions versus Enums

This proposal aims to separate out two related union-esque concepts that have been discussed for quite some time.  Instead of a single language feature that attempts to unify them into one construct, an approach is taken to keep them separated, though with one built on top of the other.

Unions would look something like:

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

To that end, to solve the latter need, this proposal recomends keeping Unions the same as currently proposed, while heavily investing on expanding existing `enums` to support this space.

### Syntactic expansion

Syntactically, the proposal suggests:

```diff
enum_declaration
-    : attributes? enum_modifier* 'enum' identifier enum_base? enum_body ';'?
+    : attributes? enum_modifier* 'enum' ('struct' | 'class')? identifier enum_base? enum_body ';'?
    ;

enum_base
-    : ':' integral_type
-    | ':' integral_type_name
+    : ':' enum_simple_type
+    | ':' enum_primitive_type_name
    ;

+ enum_simple_type
+    : simple_type | 'string' // All the primitive types that can have constant values
+    | type_name // Must resolve to any of the primitive types above.
+    ;

enum_body
-    : '{' enum_member_declarations ',' '}'
+   : '{' enum_member_declarations ',' (';' struct_member_declaration*)? '}'
    ;

enum_member_declaration
-    : attributes? identifier ('=' constant_expression)?
+    : enum_constant_value_declaration
+    | enum_shape_value_declaration
    ;

+ enum_constant_value_declaration
+    : attributes? identifier ('=' constant_expression)?
+    ;

+ enum_shape_value_declaration
+    : identifier parameter_list?
+    ;
```

## Detailed discussion

### Enum constant value declarations

First, this proposal expands primitive enums to support more than just named integral constant values.  If provided, the `enum_base` can be any C# type that we allow constants for (excluding the trivial `object + null` pairing).

Existing enums would retain their current meaning.  Being an integral backed set of values, which monotonically increase by 1 across each value (resetting if an explicit constant is provided).

Using new `enum_simple_types` (like `double`, `string`, `char`, `bool`, etc.) would require the presence of the `= constant_expression` on the individual declarations.

These constant valued enums would compile down to an  value type subclass of `System.Enum`, except with a backing `value__` field of the corrending `enum_simple_type`. 

### Enum shape declarations

The presence of `('struct' | 'class')` after the enum identifier *or* the presence of at least one `enum_shape_value_declaration` makes the `enum` a modern `enum shape`.  It is not legal to mix `enum_shape_value_declaration` and `enum_constant_value_declaration` (those with a constant initializer) in the same `enum_declaration`.

If the enum does not have `('struct' | 'class')`, but does have at least one `enum_shape_value_declaration` then it is equivalent to `enum class` (similar to how `record` is equivalent to `record class`).

An `enum shape` defines a set of constructors (and thus corresponding deconstructors) for the enum, explicitly enumerating the set of legal values the enum can have.  For example, all of the following are equivalent:

```c#
// 'class' forces this to be an enum shape.
enum class Gate { Locked, Closed } 

// Presence of parameter lists forces this to be an enum shape
enum Gate { Locked(), Closed() }
```

Note: it is fine to not have parameter lists on all elements in an `enum shape`, as long as *either* at least one declaration does have a parameter list *or* `'class' | 'struct'` is specified.  So the following is also allowed:

```c#
enum Gate { Locked, Closed, Open(float percentage) }
```

This proposal recomends that if `unions` can have members, that `enum` be allowed to have members in a similar fashion.  Note: these members would likely be restricted to either statics, or the set of instance members that do not introduce new instance state.  So no instance fields, auto-properties, etc.

