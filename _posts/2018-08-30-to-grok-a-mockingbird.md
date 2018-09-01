---
title: To Grok a Mockingbird
tags: [recursion]
---

*Using recursive combinators to enhance functional composition, with special guests the Mockingbird, Widowbird, and Sage Bird*

---

In this essay we're going to look at _recursive combinators_. A recursive combinator is a function that takes another function that is not recursive, and returns a function that is recursive. Recursive combinators make it possible to create recursive functions that are not tightly coupled to themselves.

Recursive combinators have important theoretical implications, but for the working programmer they decouple recursive functions from the mechanism that implements recursion. This makes it easier to compose recursive functions with decorators and to implement recursion strategies like trampolining.

We'll begin our exploration with a look at the *mockingbird*, also called the `M` combinator.[^little-omega] We'll then move on to examine the widowbird, a combinator that executes tail-recursive functions in constant space. We'll finish with a brief look at the famous sage bird, or Y Combinator.

[^little-omega]: The mockingbird or "M combinator" is also sometimes called ω, or "little omega". The full explanation for ω, as well as its relation to Ω ("big omega"), can be found on David C Keenan's delightful [To Dissect a Mockingbird](http://dkeenan.com/Lambda/)  page.<br/><br/>In [Combinatory Logic](https://esolangs.org/wiki/Combinatory_logic), the fundamental combinators are named after birds, following the example of Raymond Smullyan's famous book [To Mock a Mockingbird](http://www.amazon.com/gp/product/B00A1P096Y/ref=as_li_ss_tl?ie=UTF8&camp=1789&creative=390957&creativeASIN=B00A1P096Y&linkCode=as2&tag=raganwald001-20). Needless to say, the title of the book and its central character is the inspiration for this essay!

---

[![Eye in the Sky ©2011 Ian Sane](/assets/images/eye-in-the-sky.jpg)](https://www.flickr.com/photos/31246066@N04/5414394619)

---

### recursion and binding

> As the number of people discussing recursion in an online forum increases, the probability that someone will quote the definition for recursion as _Recursion: see 'recursion'_, approaches one.

This is a function that computes exponentiation. If we want to compute something like `2^8` (two to the power of eight), we can compute it like this: `2 * 2 * 2 * 2 * 2 * 2 * 2 * 2`, which requires O_n_ operations. Our function exploits basic arithmetic and recursion to obtain the same result in _O_log2n operations:[^fib]

[^fib]: This basic pattern was originally discussed in an essay about a different recursive function, [writing a matrix multiplication implementation of fibonacci](http://raganwald.com/2015/12/20/an-es6-program-to-compute-fibonacci.html).

```javascript
function exponent (x, n) {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * exponent(x * x, Math.floor(n / 2));
  } else {
    return exponent(x * x, n / 2);
  }
}

exponent(2, 7)
  //=> 128
```

Question: _How does our `exponent` function actually perform recursion?_ The immediate answer is, "It calls itself when the work to be performed is not the base case" (the base case for exponentiation is an exponent of `0` or `1`).

How does it call itself? Well, when we have a function declaration (like above), or a named function expression, the function is bound to its own name within the body of the function automatically.

So within the body of the `exponent` function, the function itself is bound to the name `exponent`, and that's what it calls. This is obvious to most programmers, and it's how we nearly always implement recursion.

But it's not _always_ exactly what we want. If we want even more performance, we might consider [memoizing](https://en.wikipedia.org/wiki/Memoization) the function.

Here's a memoization decorator, snarfed from [Time, Space, and Life As We Know It
](http://raganwald.com/2017/01/12/time-space-life-as-we-know-it.html):

```
const memoized = (fn, keymaker = JSON.stringify) => {
  const lookupTable = new Map();

  return function (...args) {
    const key = keymaker.call(this, args);

    return lookupTable[key] || (lookupTable[key] = fn.apply(this, args));
  }
};
```

We can make a memoized version of our `exponent` function:

```javascript
const mExponent = memoized(exponent);

mExponent(2, 8)
  //=> 256, performs three multiplications
mExponent(2, 8)
  //=> 256, returns the memoized result without further multiplications
```

There is a hitch with this solution: Although we are invoking `mExponent`, internally `exponent` is invoking itself directly, without memoization. So if we write:

```javascript
const mExponent = memoized(exponent);

mExponent(2, 8)
  //=> 256, performs three multiplications
mExponent(2, 9)
  //=> 512, performs four multiplications
```

When we invoke `exponent(2, 8)`, we also end up invoking `exponent(4, 4)`, `exponent(16, 2)`, and `exponent(256, 1)`. We want those memoized. That way, when we invoke `exponent(2, 9)`, and it invoked `exponent(4, 4)`, the result is memoized and it need do no further computation.

Our problem here is that `exponent` is "hard-wired" to call `exponent`, _not_ `mExponent`. So it never invoked the memoized version of the function.

We can work around that like this:

```javascript
const mExponent = memoized((x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * mExponent(x * x, Math.floor(n / 2));
  } else {
    return mExponent(x * x, n / 2);
  }
});

mExponent(2, 8)
  //=> 256, performs three multiplications
mExponent(2, 9)
  //=> 512, performs only one multiplication
```

In many cases this is fine. But conceptually, writing it this way means that our exponent function needs to know whether it is memoized or not. This runs counter to our "Allongé" style of writing things that can be composed without them needing to know anything about each other.

For example, if we wanted a non-memoized exponentiation function, we'd have to duplicate all of the code, with a minor variation:

```javascript
const exponent = (x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * exponent(x * x, Math.floor(n / 2));
  } else {
    return exponent(x * x, n / 2);
  }
};
```

That is not composing things, at all. What we want is to have one exponentiation function, and find a way to use it with or without decoration (such as with or without memoization). And we can do this.

---

[![Penrose tiling Oxford ©2014 Kelbv](/assets/images/penrose.jpg)](https://www.flickr.com/photos/flikr/14698426287)

---

### composeable recursive functions

The sticking point is that to have full memoization, our exponentiation function needs to have a hard-coded reference to the memoized version of itself, which means it can't be used without memoization. This is a specific case of a more general problem where things that have hard-coded references to each other become tightly coupled, and are thus difficult to compose in different ways. Only in this case, we've made the thing tightly coupled to itself!

So let's attack the hard-coded reference problem, decoupling our recursive function from itself. Since it doesn't have to be a named function, we can make it a "fat arrow" expression. If we want a function to have a reference to another function in JavaScript, we can pass it in as a parameter. So the 'signature' for our new function expression will look like this:

```javascript
(myself, x, n) => // ...
```

In this case, our function assumes that `myself` is going to be bound to the function itself. Now what about the body of the function? We can change `exponent` to `myself`:

```javascript
(myself, x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * myself(x * x, Math.floor(n / 2));
  } else {
    return myself(x * x, n / 2);
  }
};
```

One little hitch: Our function signature is `(myself, x, n)`, but when we invoke `myself`, we're only passing in `x` and `n`. So we can pass `myself` in as well:

```javascript
(myself, x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * myself(myself, x * x, Math.floor(n / 2));
  } else {
    return myself(myself, x * x, n / 2);
  }
};
```

Now this seems very contrived, and it doesn't even work yet. How can we make it work?

---

[![Galápagos Mockingbird ©2012 Ben Tavener](/assets/images/galapagos-mockingbird.jpg)](https://www.flickr.com/photos/bentavener/7137047259)

---

### the mockingbird

Behold, the JavaScript mockingbird:

```javascript
const mockingbird = fn => (...args) => fn(fn, ...args);
```

The mockingbird is a function that takes another function, and returns a function. That function takes a bunch or arguments, and invoked the original function with itself and the arguments.[^well-actually] So now we can write:

[^well-actually]: In proper combinatorial logic, the mockingbird is actually defined as `M x = x x`. However, this presumes that all combinators are "curried" and only take one argument. Our mockingbird is more "idiomatically JavaScript."<br/><br/>But it's certainly possible to use `const M = fn => fn(fn);`, we would just need to also rewrite our exponentiation function to have a signature of `myself => x => n => ...`, and so forth. That typically clutters JavaScript up, so we're using `const mockingbird = fn => (...args) => fn(fn, ...args);`, which amounts to the same thing.

```javascript
mockingbird((myself, x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * myself(myself, x * x, Math.floor(n / 2));
  } else {
    return myself(myself, x * x, n / 2);
  }
})(2, 8)
  //=> 256
```

That is all very well and good, but we've added some extra bookkeeping. Do we have any wins? Let's try composing it with the memoization function. Although we didn't use it above, our `memoize` function does allow us to customize the function used to create a key. Here's a key making function that deliberately ignores the first argument:

```javascript
const ignoreFirst = ([_, ...values]) => JSON.stringify(values);
```

And now we can create a memoized version of our anonymous function. First, here it is step-by-step:

```javascript
const exp = (myself, x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * myself(myself, x * x, Math.floor(n / 2));
  } else {
    return myself(myself, x * x, n / 2);
  }
};

mockingbird(memoized(exp, ignoreFirst))(2, 8)
  //=> 256
```

But now for the big question: Does it memoize everything? Let's test it:

```javascript
const mExponent = mockingbird(memoized(exp, ignoreFirst));

mExponent(2, 8)
  //=> 256, performs three multiplications
mExponent(2, 9)
  //=> 512, performs only one multiplication
```

Yes it does properly memoize everything. And best of all, our function need have absolutely NO reference to the name of our memoized function.[^aha] It doesn't know whether it's memoized or not.

[^aha]: In JavaScript, like almost all programming languages, we can bind values to names with paramaters, or with variable declarations, or with named functions. So having something like the M Combinator is optional, as we can choose to have a function refer to itself via a function name or variable binding. However, in Combinatory Logic and the Lambda calculus, there are no variable declarations or named functions.<br/><br/>Therefore, recursive combinators are necessary, as they are the only way to implement recursion. And since they don't have iteration either, recursion is the only way to do a lot of things we take for granted in JavaScript, like mapping lists. So recursive combinators are deeply important to the underlying building blocks of computer science.

Because we've separated the function from the mockingbird that implements recursion, we can compose our exponentiation function with memoization or not as we see fit. When the exponentiation function was responsible for directly calling itself, if we wanted one version memoized and one not, we'd have to write two nearly identical versions of the same code.

But with the mockingbird separating how a function calls itself from the function, we can now write:

```javascript
const exp = (myself, x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * myself(myself, x * x, Math.floor(n / 2));
  } else {
    return myself(myself, x * x, n / 2);
  }
};

const mExponent = mockingbird(memoized(exp, ignoreFirst));
const exponent = mockingbird(exp);
```

We have our composeability and reuse!

---

[![Chrysler Imperial ©2017 pyntofmyld](/assets/images/chrysler-imperial.jpg)](https://www.flickr.com/photos/pyntofmyld/24408966088)

---

### tail recursion

Here's a naive version of our exponentiation function. Not only does it perform O_n_ operations (instead of _O_log2n operations), but its use of recursion is completely gratuitous. But we'll experiment with it, as it provides a good demonstration of the perils of deeply recursive functions.

We've written it in "mockingbird" form:

```javascript
const naive = (myself, x, n) => {
  if (n === 0) {
    return 1;
  } else {
    return x * myself(myself, x, n - 1);
  }
}

naive(2, 10)
  //=> 1024
```

Many recursive functions can be rewrittten in "tail recursive form" Here it is again, with its recursive call "in tail position:"[^tail]

[^tail]: See [A Trick of the Tail](http://raganwald.com/2018/05/27/tail.html) for a fuller explanation of how to perform this refactoring.

```javascript
const naive = (myself, x, n, acc = 1) => {
  if (n === 0) {
    return acc;
  } else {
    return myself(myself, x, n - 1, x * acc);
  }
}

mockingbird(naive)(2, 10)
  //=> 1024
```

What happens if we write:

```javascript
mockingbird(naive)(1, 1000000)
```

We know the answer is `1`, but do we get it? If we evaluate the above on a JS engine that supports tail call optimization, we get it. But on other implementations, we get `Maximum call stack size exceeded`.[^tco]

[^tco]: Our code works just fine on the Safari browser, which in addition to being far more thrifty with battery life on OS X and iOS devices, implements Tail Call Optimization, as specified in the JavaScript standard. Alas, most other implementations refuse to implement TCO.

As discussed in [Trampolines in JavaScript][trampoline], we can get around the call stack problem ourselves with a technique called trampolining:

[trampoline]: http://raganwald.com/2013/03/28/trampolines-in-javascript.html

> A trampoline is a loop that iteratively invokes [thunk]-returning functions ([continuation-passing style][cps]). A single trampoline is sufficient to express all control transfers of a program; a program so expressed is trampolined, or in trampolined style; converting a program to trampolined style is trampolining. Trampolined functions can be used to implement [tail-recursive] function calls in stack-oriented programming languages.--[Wikipedia][trampolining]

[thunk]: https://en.wikipedia.org/wiki/Thunk_(functional_programming)
[cps]: https://en.wikipedia.org/wiki/Continuation-passing_style
[tail-recursive]: https://en.wikipedia.org/wiki/Tail-recursive_function
[trampolining]: https://en.wikipedia.org/wiki/Trampoline_(computing)

Trampolining functions are straightforward to write, but the typical pattern has a drawback. From the OP on the subject, here's `factorial` written to work with the trampolining function, updated for contemporary JavaScript:

```javascript
class Thunk {
  constructor (delayed) {
    this.delayed = delayed;
  }

  evaluate () {
    return this.delayed();
  }
}

const trampoline =
  fn =>
    (...initialArgs) => {
      let value = fn(...initialArgs);

      while (value instanceof Thunk) {
        value = value.evaluate();
      }

      return value;
    };


function factorial (n) {
  const _factorial = trampoline(
    function myself (n, acc = 1) {
      return n
        ? new Thunk(() => myself(n - 1, acc * n))
        : acc
    }
  );

  return _factorial(n);
}

factorial(5)
  //=> 120
```

This works, but again we have the factorial function coupled to itself, _and_ the inner recursive function is coupled to the implementation of trampolining. Why should it know what a thunk is?

---

[![Mockingbird nabs a berry ©2011 Kim Taylor](/assets/images/mockingbird-berry.jpg)](https://www.flickr.com/photos/kim/6371285287)

---

### a new kind of passerine science

A mockingbird cannot directly solve our problem, but we can learn from the mockingbird to write a new kind of trampolining function based on the mockingbird. We start by rewriting our factorial function in both tail-recursive and decoupled form. Note that this works just fine with our mockingbird:

```javascript
const factorial =
  (myself, n, acc = 1) =>
    n
      ? myself(myself, n - 1, acc * n)
      : acc

mockingbird(factorial)(5)
  //=> 120
```

Now we've decoupled the form of the function from the mechanism of recursion. So, let's swap the mechanism of recursion for a trampoline _without altering the recursive function to suit the new implementation_.

We'll call our new "combinator" a Widowbird:[^widowbird]

[^widowbird]: The Jackson's Widowbird, _Euplectes Jacksoni_, is a passerine bird in the family Ploceidae. As notably portrayed in BBC Planet Earth II, when attempting to attract females to nest in their territory, the males repeatedly jump to show off their fitness. If we exercise our vivid imaginations, we can think of this as resembling the behaviour of a trampolining tail-recursive function. Instead of "drilling deeper and deeper," it repeatedly bounces back up to the top.

```javascript
const widowbird =
  fn => {
    class Thunk {
      constructor (args) {
        this.args = args;
      }

      evaluate () {
        return fn(...this.args);
      }
    }

    return (...initialArgs) => {
      let value = fn(
        (...args) => new Thunk(args),
        ...initialArgs
      );

      while (value instanceof Thunk) {
        value = value.evaluate();
      }

      return value;
    };
  };

widowbird(factorial)(5)
  //=> 120
```

Since we're passing the function to be called recursively into our recursive function, we can place the thunk mechanism in our `widowbird`. Thus, the recursive function is completely decoupled from the mechanism for recursing without consuming the stack.

And what about our `naive` exponentiation that broke the stack earlier?

```javascript
widowbird(naive)(1, 1000000)
  //=> 1
```

It works just fine, even on engines that don't support tail call optimization. The widowbird has shown us another benefit of separating the recursive computation to be done from the mechanism for performing the recursion.

---

[![y? ©2012 Newtown grafitti](/assets/images/y.jpg)](https://www.flickr.com/photos/newtown_grafitti/8286552835/)

---

### the sage bird

The mockingbird has the advantage of being the very simplest recursive combinator. But it can be enhanced. One of the annoying things about it is that when we write our functions to use with a mockingbird, not only do we need a `myself` parameter, but we need to remember to pass it on as well.

This isn't a bad tradeoff, but logicians searched for a combinator that could implement recursion with a parameter, like the mockingbird, but avoid having to pass that parameter on. This had important theoretical consequences, but for us, the value of such a combinator is that the functions we write are more natural.

The combinator that decouples recursion using a parameter, but doesn't require passing that parameter along, is called the [Y Combinator][y-combinator], or _Sage Bird_.

[y-combinator]: https://en.wikipedia.org/wiki/Fixed-point_combinator

A simple and direct JavaScript implementation looks like this:

```javascript
const Y = fn =>
  ( x => fn(v => x(x)(v)) )(
    x => fn(v => x(x)(v))
  );
```

Without getting into exactly how it works,[^whyy] we can see that the disadvantage of the direct implementation is that once again, it assumes that all functions are curried to take only one argument. Here's an idiomatic JavaScript version, one that handles functions with more than one argument. It is written to preserve the form of the direct implementation:

[^whyy]: There are lots of essays deriving the Y Combinator step-by-step. Here's one in [JavaScript](https://enlight.nyc/y-combinator/), and here's [another](http://igstan.ro/posts/2010-12-01-deriving-the-y-combinator-in-7-easy-steps.html).

```javascript
const sagebird =
  fn => (
    (innerFn) =>
      ( myself => innerFn((...args) => myself(myself)(...args)) )(
        myself => innerFn((...args) => myself(myself)(...args))
      )
  )(myself => (...args) => fn(myself, ...args));
```

Armed with our sage bird, we can write recursive functions that look a little more idiomatic:

```javascript
const exp = (myself, x, n) => {
  if (n === 0) {
    return 1;
  } else if (n % 2 === 1) {
    return x * myself(x * x, Math.floor(n / 2));
  } else {
    return myself(x * x, n / 2);
  }
};

sagebird(exp)(2, 10)
  //=> 1024
```

No more `myself(myself, x * x, n / 2)`!

The sage bird has more complicated workings, but it makes the code we write much simpler. And like the mockingbird, it allows us to separate the mechanism for recursion from the function we wish to make recursive.

---

[![The Summary Key ©2017 Mike Lawrence](/assets/images/summary.jpg)](https://www.flickr.com/photos/157270154@N05/38494483572)

---

### summary

In summary, the mockingbird is a _recursive combinator_: It takes a function that is not directly recursive, and makes it recursive by passing the subject function to itself as a parameter. This has the effect of removing a hard-coded dependency between the subject function and itself, which allows us to decorate it with functionality like memoization.

We've also seen that having performed this separation, we can swap the mockingbird out for other functions implementing recursion, such as the widowbird. We've seen that the widowbird is superior to other approaches, because it does not require the function being trampolined to "know" that it is being trampolined.

And finally, we saw the sage bird, or Y Combinator. We saw that it makes our functions a little more idiomatic, and once again delivers the value of separating function from recursion mechanism.

Recursive combinators like mockingbirds, widowbirds, and sage birds are a few more tools in our "composeable functions" toolbox, increasing reuse by decoupling recursive functions from themselves.

(discuss on reddit [here](https://www.reddit.com/r/javascript/comments/9bu5od/to_grok_a_mockingbird_using_recursive_combinators/), or [here](https://www.reddit.com/r/programming/comments/9bvmls/to_grok_a_mockingbird_using_recursive_combinators/), or on [hacker news](https://news.ycombinator.com/item?id=17885852))

---

## Notes

