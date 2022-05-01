---
layout: post
title: TypeScript is not as Sound as You Think
---

The point of this blog post is to highlight that programming language design is not a solved problem and there are unexpected issues in highly popular languages that remains unsolved of which you should be aware of.

## Introduction

When we programmers write code we trust the compiler to do its job. A compiler has many tasks. Roughly these include and are performed in the following steps: first it will try to verify that the source code is correct, then convert human-readable source code to something more machine-like, optimizing the generated code for speed and finally producing an executable binary. A "correctness" verification, or semantic checking, can only go so far, as the compiler does not know what you really want to achieve with your code. But, at a minimum it will check that you are syntactically correct, e.g. are you are using words defined in an English dictionary? 

> calm restaurant crop dark cruelty

These five words exists in an English dictionary, but does not form a valid sentence. The next step in semantic checking is to verify that words are used in a particular order according to some defined grammar.

> One plus one is five.

This is a grammatically correct sentence in English, but its semantics (i.e. meaning) is logically incorrect. It would be up to the language designers to define if the compiler should try to catch these errors (if it's possible) or if the programmer should have more expressive freedom. 

How can we give hints about what the code is supposed to achieve, so that the compiler can try to check for at least some set of logical errors? Well, part of the answer is to statically define types. In TypeScript we use types, which then can catch semantic errors like so:

{% highlight typescript linenos %}
const a: number = 1;
const b: number = 1;
const c: string = a + b;
{% endhighlight %}

Since we know that `a` and `b` are a `number` the addition of two `numbers` should still be a `number`. Thus, the assignment to `c` is therefore illegal in TypeScript. In another universe the language designers might have allowed this scenario by saying that if you assign a number to a string, implicitly convert it to its string representation, but the TypeScript designers thought otherwise. Notably, in JavaScript this is valid code and is what they would do. The up-side is maximum expressiveness but at the cost of the compiler being less able to identify errors.

Now, I started this post with saying that we are trusting the compiler to do its job. Particularly we assume that since we spend a lot of time writing type information in TypeScript (or any other statically typed language for that matter), we assume that any logical error that can be deduced from types are identified. It turns out that the type checker is not perfect. Maybe not so unsurprisingly, we will see that TypeScript (which are fully compatible with JavaScript) suffers from inconsistencies. In the research field of programming languages we say that a type system is sound iff:

> `If x has type ` \\( \tau \\) `, then either`
>
> `x is a value, or`
>
> `x can take a step to some x' which also has type `\\( \tau \\)

This means that a type checker with its defined semantics that fulfills the above will identify programs that does not adhere to the defined semantics.

## Unsoundness in TypeScript
TypeScript does not even try to be sound, since one design goal with TypeScript is to be fully compatible with JavaScript. Since JavaScript is not statically typed it is by design not sound, and as such TypeScript will suffer from this as well. The goal with TypeScript is to do a best-effort static analysis. Still, the type checker may fail to catch errors in surprising ways.

{% highlight typescript linenos %}
const x: {f: string} = {f: "hi"};
const y: {f: number | boolean} = {f: 42};

var z;
if (Math.random() < 0.5) {
  z = x;
} else {
  z = y;
}

if (typeof(z.f) === "number") {
  z.f = "broken";
} else {
  z.f = 111;
}

console.log(z);
{% endhighlight %}

The type of `z` is derived from `x` and `y` to be `{f: string | number | boolean}`. Even if TypeScript have all type information to either warn or throw an error it happily compiles this code without any issue, due to the way it derives the type of `z`. But the world is not broken due to type of `z`, but rather what it allows us to do with the original reference that `z` references. If `z.f` is a string you can't write a number since that would break the type definition of `x`. Regrettably, this is exactly what happens and TypeScript does not complain. The world is thus broken!

_Thanks to Elias and Ellen for pointing out this example_
