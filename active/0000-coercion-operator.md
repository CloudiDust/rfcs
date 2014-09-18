- Start Date: 2014-09-18
- RFC PR: (leave this empty)
- Rust Issue: (leave this empty)

# Summary

Reintroduce `~` as the semi-explicit coercion operator, so coercions in Rust can be both ergonomic and explicit enough. 

# Motivation

Recently we are talking about introducing more implicit coercions into the language:

[RFC PR 225](https://github.com/rust-lang/rfcs/pull/225): Add safe integral auto-coercions.
[RFC PR 226](https://github.com/rust-lang/rfcs/pull/226): Allow implicit coercions for cross-borrows.
[RFC PR 241](https://github.com/rust-lang/rfcs/pull/241): Deref coercions.

Also there are discussions about introducing a [`Coerce` trait](http://discuss.rust-lang.org/t/pre-rfc-add-a-coerce-trait-to-get-rid-of-the-as-slice-calls/415/8).

That's because, while Rust prefers to be explicit, there are times when being too explicit can be ergonomic problems, for example:

Given a function `fn foo(bar: &T)`, and a value `fancy: Very<Deep<Chain<Of<Wrappers<T>>>>>`, how to apply `foo` to the wrapped `T` inside `fancy`?

Currently, `fancy` has to be dereferenced then referenced explicitly, by calling `foo(&*****fancy)`. This is not pretty, even the more common `&*foo`'s are less than ideal.

So implicit coercions are proposed. With the rules in RFC PR 241, simply calling `foo(fancy)` would be enough. This is much better.

But implicit coercions also have their own problems. The main one being that, innocent looking code may not do what the programmer expects them to do. For example, what does the function call `foo(bar, baz)` do to its arguments? Does it simply move or copy `foo`/`bar`? Or take references to something wrapped in `foo`/`bar`? Or do other arbitrary things even before executing the function body?

If implicit coercions are disallowed, then surly the answer is "it simply moves or copies `foo`/`bar`". But with implicit coercions, all bets are off. It'll be harder for the programmer to reason about code.

So, being too explicit and too implicit are both undesirable sometimes, then what to do?

Observation: **Often, when a programmer writes `&*`s and explicit `as` coercions, he/she doesn't really *need* to care about exactly how many `*`s or which type he/she has to write. To efficiently reason about code, knowing that "some coercion magic happens here" is often enough.**

Thus, the trick is: **Be explicit, but not too explicit.**

# Detailed design

Reintroduce `~` as the semi-explicit coercion operator.

## Syntax Design:

1. `~` will be reintroduced as a *postfix unary operator*, i.e. `foo~`;
2. `foo~?` means `(foo~)?` where `?` is any postfix unary operator, including another `~`;
3. Postfix unary operators take precedence over prefix ones, which means `?foo~` is interpreted as `?(foo~)` where `?` is any prefix unary operator.

These rules are to keep `~` in synchronization with the postfix unary operators `?`/`!` in [RFC PR 204](https://github.com/rust-lang/rfcs/pull/225) and [RFC PR 243](https://github.com/rust-lang/rfcs/pull/243)

Note: `foo~?`, `(foo~)?`, `?foo~` and `?(foo~)` are all valid syntactically, but invalid semantically. Please refer to the discussions below.

## Semantics Design:

First, most implicit coercions should be disabled:

1. Rust the language can define arbitrary coercion rules between types, which may or may not be user-defined, but none of the rules is applicable implicitly.
2. There is one and only one exception to the above rule, the semantics of the field accessing/method calling notation `foo.bar` remains the same as today.

Then, the semantics of `~`:

For any expression `expr` of type `T`, the expression `expr~` can be written at locations in source code `CL` if and only if:
1. `CL` expects a expression of type `U`, which is distinct from `T`.
2. Of all possible coercion rules that coerces a value of `T` to a value of `U`, exactly one rule `CR` is potentially applicable at CL.
3. `CL` is not a `self` position of a field access or a method call.

The expression `expr~` cannot be written anywhere else, and at runtime, `expr~` at `CL` evaluates to `CR(expr)`.

Note: The semantics here speaks of multiple coercion rules that all do `T -> U`, and this is only for the sake of completeness. Importable user-definable coercion rules and disambiguations between conflicting coercion rules are outside the scope of this RFC.

The semantics of `~` effectively means that the compiler can infer applicable coercions for the programmer, but the programmer must opt-in this behaviour with `~` on a case-by-case basis, otherwise he/she will get compile errors. And if no coercions are applicable, `~` cannot be used.

`~` is a special operator, as what `expr~` evaluates to depends on its location. As far as the parser is concerned, `~` is just a normal postfix unary operator, and notations like `foo~~` are valid. But `foo~~` will get rejected in the semantics analysis stage, as `foo~~` means `(foo~)~`, and the first `~` is in a location that can accept any expression, so the compiler cannot find a coercion rule to apply here, resulting in a compile error.

In general, (TODO: further discussions.)

## Examples:

For the example in **Motivation**, we can say `foo(fancy~)`.

Additional examples:

### Deep dereferences and cross-borrowing:

### Safe integer coercing:

### Trait object coercing:

# Drawbacks

1. `~` is yet another sigil.
2. `~` doesn't exist on some keyboard layouts.
3. `~` is a bit special as an operator.
4. `~` still makes some coercions not explicit enough.

# Alternatives

1. Prefix `~`.
2. `!` instead of `~`.
3. Implicit coercions, or fully explicit coercions.
4. Repurpose `&` to be a borrow operator.
5. Further replace `as` with binary `~`.

# Unresolved questions
