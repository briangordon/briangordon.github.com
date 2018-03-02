---
layout: post
title: Square roots and fixed points
tags: programming math
published: true
---

There's [an old HN comment](https://news.ycombinator.com/item?id=571090) in which a developer whines bitterly and eloquently about his experience being stumped by a common interview question: "write an algorithm for computing the square root of a number." 

> I tried a couple more times to answer this question that is completely unrelated to the job I'm interviewing for. Finally, I get so fed up with this moron that my internal ticker clicks over and I realize, "Even if I get this job, I'm going to be dealing with these kinds of nazel-gazing engineers every single day. Not an environment I want to be in."

![OK](https://i.imgur.com/seh6p.gif)

The most obvious solution is binary search, but that's not very interesting. How else might we solve the problem of calculating $$\sqrt{a}$$? Let's first introduce the concept of a **fixed point**. A fixed point is a value which is unchanged by the function - that is, $$f(p) = p$$. For example, a fixed point of the sine function is 0 because $$\sin(0)=0$$. A fixed point of the cosine function is located around 0.739085133 because $$\cos(0.739085133)\approx0.739085133$$. In fact, if we plot the cosine function on top of $$f(p) = p$$ then we can see that they intersect at exactly that point:

![Plot 1](/images/plot-0.svg)

One interesting fact which might be of use is that $$\sqrt{a}$$ is a fixed point of the function $$f(x)=\frac{a}{x}$$:

$$ \begin{eqnarray*}
f(\sqrt{a}) &=& \frac{a}{\sqrt{a}}\\
            &=& a^1a^{-\frac{1}{2}}\\
            &=& \sqrt{a} 
\end{eqnarray*} $$

We can therefore graph $$f(x)=\frac{a}{x}$$ to find its fixed point $$\sqrt{a}$$. Here's a plot for various values of a:

![Plot 2](/images/plot-1.svg)

By subtracting x from the function it becomes a root finding problem:

![Plot 3](/images/plot-2.svg)

This we can solve with Newton's method:

$$ x_{n+1} = x_n - \frac{f(x_n)}{f'(x_n)} $$

$$ x_{n+1} = x_n - \frac{\frac{a}{x_n}-x_n}{-\frac{a}{x^{2}}-1} $$

$$ x_{n+1} = \frac{2x_n{}^3}{x_n{}^2+a} $$

Of course, we could have just used Newton's method from the beginning. If we're trying to calculate $$\sqrt{a}$$ then the value we're looking for is a zero of the function $$f(x)=x^{2}-a$$. In this case we have:

$$ x_{n+1} = x_n - \frac{f(x_n)}{f'(x_n)} $$

$$ x_{n+1} = x_n - \frac{x_n{}^2-a}{2x_n} $$

$$ x_{n+1} = \frac{x_n{}^2+a}{2x_n} $$

These are different functions but they have the same fixed point. This is a plot of $$ x_{n+1} - x_n $$ for the two different functions (with a=7) showing that they have the same zeros:

![Plot 4](/images/plot-3.svg)

Newton's method corresponds closely to a concept called **fixed point iteration**. In fixed point iteration, we repeatedly evaluate a function and feed its output back into its input. Eventually we hope it will converge on a fixed point. We mentioned the cosine function earlier; this is one such function which always converges on a particular fixed point. Enter any value into a scientific calculator and repeatedly press the cosine button:

<p style="text-align:center">-1.0000, 0.5403, 0.8575, 0.6542, 0.7934, 0.7013, 0.7639, 0.7221, 0.7504, 0.7314, 0.7442, 0.7356</p>

It will eventually converge to a value near 0.7391:

![Plot 5](https://upload.wikimedia.org/wikipedia/commons/e/ea/Cosine_fixed_point.svg)

Another function which converges to a fixed point is the expression given by Newton's method applied to the function $$f(x)=x^2-a$$, which we derived before:

$$ x_{n+1} = \frac{x_n{}^2+a}{2x_n} $$

If we rearrange the right side we obtain:

$$ x_{n+1} = \frac{1}{2}\left(x_n+\frac{a}{x_n}\right) $$

This is called the Babylonian method, or Heron's method, and it was actually discovered long before Newton's method. The idea was that if $$x_n$$ is an overestimate to $$\sqrt{a}$$ then $$\frac{a}{x_n}$$ is an underestimate, and you can average them to get a better approximation. Repeated iteration improves this approximation until it converges on $$\sqrt{x}$$, which we know is a fixed point.

Remember that before we said $$\sqrt{a}$$ is a fixed point of the function $$f(x)=\frac{a}{x_n}$$. Unfortunately if you iterate that function, you will not approach $$\sqrt{a}$$. Fixed point iteration doesn't always work and this is one such case. The math behind being able to tell whether an arbitrary function will converge to a fixed point under fixed point iteration is [complicated](https://en.wikipedia.org/wiki/Fixed-point_theorem). 

Now it would be very simple to wrap the Babylonian method in a loop and perform a couple steps of fixed point iteration to get a decent sqrt(a). But since we're finding a fixed point, this seems like a nice time to break out something called a *fixed point combinator*. The best-known fixed point combinator is the *Y combinator*. You've probably heard of it due to [the eponymous startup incubator](https://en.wikipedia.org/wiki/Y_Combinator_(company)) founded by Lisp greybeard Paul Graham, of the famous [Paul Graham essays](http://paulgraham.com/articles.html).

![YCombinator logo](https://old.ycombinator.com/images/yc500.gif)

This is the [lambda calculus](https://en.wikipedia.org/wiki/Lambda_calculus) definition of the Y combinator, due to Haskell Curry:

$$ \lambda f.\!(\lambda x.f\ (x\ x))\ (\lambda x.f\ (x\ x)) $$

The reason y is called a fixed-point combinator is because of what happens when you apply it to a function and reduce. By following the lambda calculus reduction rules you can find that the Y combinator satisfies the equation $$y\ f = f\ (y\ f)$$. This matches the form $$something = f\ (something)$$ - the definition of a fixed point. So, $$y\ f$$ is a fixed point of $$f$$! Therefore all we have to do is apply the Y combinator to obtain a fixed point of $$f$$, right?

Well, not really. $$f$$ takes and returns a function so the fixed point of $$f$$ isn't a number at all, it's the function $$f'$$ which $$f$$ maps to $$f'$$. In other words, all the Y combinator is doing is facilitating fixed-point iteration:

$$ \begin{eqnarray*}
y\ f &=& f\ (y\ f)\\
     &=& f\ (f\ (y\ f))\\
     &=& f\ (f\ (f\ (y\ f)))\\
     &=& f\ (f\ (f\ (f\ (y\ f))))\\
     && ... 
\end{eqnarray*}$$

It appears that this expansion will continue forever and never terminate. But we can build a termination condition into the function $$f$$ so that it stops expanding. Let's see how that would work with the sqrt example. Our $$f$$ could look like this in JavaScript:

~~~ javascript
function step (callback) {
    return function (originalValue, approxSqrt) {
        // Babylonian method formula to improve our approximation of the square root 
        // of originalValue.
        var improvedApproxSqrt = (approxSqrt + (originalValue / approxSqrt)) / 2;

        // How far off the mark we are.
        var discrepancy = Math.abs(originalValue - 
                                (improvedApproxSqrt * improvedApproxSqrt));

        // Termination condition
        if(discrepancy < 0.00001) {
            return improvedApproxSqrt;
        }

        return callback(originalValue, improvedApproxSqrt);
    };
}
~~~

This looks a lot like recursion, except that we've never referred to an free variable name. This is called [anonymous recursion](https://en.wikipedia.org/wiki/Anonymous_recursion), and it's useful in systems (notably lambda calculus) where functions cannot refer to themselves by name.

Now we need the magic which will repeatedly invoke `step` and feed its output back into its input: the Y combinator. How would it look in JavaScript? Here's the Y combinator in lambda calculus again:

$$ \lambda f.\!(\lambda x.f\ (x\ x))\ (\lambda x.f\ (x\ x)) $$

This corresponds pretty directly to a JavaScript function:

~~~ javascript
function Y(f) {
    return (function (x) {
        return f(x(x));
    })(function (x) {
        return f(x(x));
    });
}
~~~

Unfortunately, when we try to use this version of the Y combinator, we get a stack overflow. If you trace out the execution you'll see that `x(x)` must be evaluated in order to get a final return value for `Y`, and this causes infinite recursion. The reason this works at all in lambda calculus is that lambda calculus is [call by name](https://en.wikipedia.org/wiki/Call_by_name) so $$f (x\ x)$$ is evaluated by expanding the *definition* of $$x\ x$$ and passing that function to f. JavaScript, on the other hand, is [call by value](https://en.wikipedia.org/wiki/Call_by_value), so the x function is *actually evaluated* with x as an argument. 

If we η-reduce the Y combinator then we obtain an alternate fixed-point combinator, called the Z combinator, which contains an extra layer of indirection and prevents runaway recursion:

$$ \lambda f.\!(\lambda x.f\ (\lambda v.\!((x\ x)\ v)))\ (\lambda x.f\ (\lambda v.\!((x\ x)\ v))) $$

Here's the JavaScript code corresponding to the Z combinator:

~~~ javascript
function Z(f) {
    return (function (x) { 
        return f(function (v) { return x(x)(v); }); 
    })(function (x) {
        return f(function (v) { return x(x)(v); });
    });
}
~~~

We still have a bit of a problem: our step function takes two variables, while Z only calls it with one. So let's modify Z:

~~~ javascript
function Z(f) {
    return (function (x) { 
        return f(function (v1, v2) { return x(x)(v1, v2); }); 
    })(function (x) {
        return f(function (v1, v2) { return x(x)(v1, v2); });
    });
}
~~~

Now we can see it working:

~~~
> var sqrt = Z(step);
> sqrt(2, 1); // sqrt of 2, with a starting estimate of 1
1.4142156862745097
~~~

It's rather awkward to always have to provide a starting estimate, so we can add a wrapper which always guesses 1 to start:

~~~
> function sqrt (num) { return Z(step)(num, 1); }
> sqrt(2)
1.4142156862745097
~~~
