---
layout: post
title: Developer interview questions
tags: programming math
published: true
---

The following is my attempt to write down some of the stuff that I've read and thought about when trying to come up with interview questions to ask software engineer applicants. Some of these questions are pretty hard and inappropriate for most interviews. You may nevertheless find it interesting to challenge yourself with them.

***Prompt:** Say you're working for a shipping operation that moves bulk commodities across the ocean. You can select from an exchange which goods you want to accept for delivery on a given ship. Each lot on the exchange is for a certain amount of money to ship a certain number of cubic feet of a given good. How would you decide what lots to carry on a ship with a known displacement and volume capacity?*

**Answer:** If you only had to consider volume capacity then you might be tempted to take the lots which yield the most money per cubic foot, in order, until your ship is full. Even in this simplified case, that wouldn't be an optimal solution. You might greedily take a single lot which is slightly more profitable than the others per cubic foot, but leaves 4% of your ship empty, which you cannot fill since the other lots are for at least 5% of your ship. It could be more profitable overall to accept smaller lots so that your ship can be packed more completely. This sounds like a bin-packing problem, so we know at this point that our algorithm is probably going to be exponential in the worst case.

The prompt mentions displacement, which adds another constraint to our choice of lots. It might be very profitable to ship gold bullion, but if you filled your entire ship with gold then it would sink! This is an elementary integer programming problem. Your variables are booleans signifying whether or not to accept each lot on the exchange. Your objective function is the total money made by shipping the lots that you accept. You are constrained by the total weight and volume of those lots. And once you've found the optimal shipment you should make sure that it's actually profitable given your costs!

The literature on solving integer programming problems is rich and there are many sophisticated optimizations to be made. It should be sufficient to pull an existing library off the shelf once you have the problem set up.

Note that it's probably infeasible to solve the very large systems that you could construct if there are many available lots on the exchange. Integer programming has no known polynomial-time solution. To improve performance you can relax your boolean constraints to continuous ones and consider the problem as a linear programming problem, rounding your answer for each variable. This wouldn't give a perfectly optimal solution but it could be solved in polynomial time. You could also eliminate some lots from consideration due to other lots being obviously more profitable.

***Prompt:** Given a number's prime factorization, how many distinct divisors does that number have? Why do only perfect squares have an odd number of distinct divisors?*

**Answer:** To count the number of distinct divisors, consider a number's prime factorization:

$$x = a_1^{b_1} \cdot a_2^{b_2} \cdot \ldots \cdot a_n^{b_n}$$

We know that every divisor of $$x$$ can be written uniquely as:

$$a_1^{c_1} \cdot a_2^{c_2} \cdot  \ldots \cdot a_n^{c_n}\ \vert\ 0 \leq c_i \leq b_i$$

In other words, we have $$b_1 + 1$$ choices (zero through $$b_1$$) for the first factor's exponent, $$b_2 + 1$$ choices for the second factor's exponent, and so on. The total number of divisors is therefore:

$$\displaystyle \prod_{i=1}^{n} (b_i + 1)$$

To see why only perfect squares have an odd number of divisors, think about a regular old number $$n$$ that's not a perfect square. You can pair all of its divisors $$d$$ with another divisor $$n / d$$. For example, if $$n = 12$$ then there are three pairs of divisors: $$\newcommand{\tuple}[1]{\left \langle #1 \right \rangle} \tuple{1, 12}$$, $$\tuple{2, 6}$$, and $$\tuple{3, 4}$$. Since all of the divisors are paired up, the number of divisors must be even. However, if $$n$$ is a perfect square, then one of the divisors can't be paired. The other divisors can be paired like before, but $$\sqrt{n}$$ "pairs" with itself, making an odd number of distinct divisors.

***Prompt:** You're tasked with creating a machine for automatically making change. Your machine should take as an input the amount of change to make (e.g. 16.00). It should output the number of each coin to dispense so that the fewest possible coins are dispensed. Your machine should work with any currency's coin denominations.*

**Answer:** The change-making algorithm that cashiers in the United States follow is simple. You take as many quarters as you can without going over the desired amount, then you do the same for dimes, then nickels, then pennies. You consider each coin in descending order of value. However, this greedy algorithm doesn't work in general. If there's a 6-cent coin, 18 cents should be given as three 6-cent coins, not a dime, a 6-cent coin, and two pennies. 

This is a classic dynamic programming problem. The first step in the dynamic programming approach is to find a recurrence. Let's define $$C(x)$$ to be the number of coins that it takes to make $$x$$ cents. We can define this recursively with a case for each coin. Say the denominations are 1, 4, and 5.  

$$C(X)=min\begin{cases}
C(X-1)+1\ (\text{if}\ X\ge1)\\
C(X-4)+1\ (\text{if}\ X\ge4)\\
C(X-5)+1\ (\text{if}\ X\ge5)\\
\end{cases}$$

For example, if we have to make change for 55 cents, then we can either 1) make 54 cents and add a 1 cent coin, 2) make 51 cents and add a 4 cent coin, or 3) make 50 cents and add a 5 cent coin. Our base case is that 0 cents can be made with 0 coins.

However, writing this code as a recursive function would be a terrible idea. There's an enormous amount of overlap in the computation required to evaluate each case of the min. If we tried to make change for just 50 cents, we'd end up evaluating our function 818,598,723 times. 

We could cache the result for each x after it's evaluated and check the cache before recursing (this is called memoization) but it would be better to write the algorithm iteratively. 

Say we need to make change for 100 cents. Then we're going to fill in an array of 100 entries. The first entry will represent how many coins it takes to make 1 cent, the second entry will represent how many coins it takes to make 2 cents, and so on. We can evaluate each entry in sequence using the recurrence and the entries that we've already evaluated. After 100 steps we'll have evaluated how many coins it takes to make 100 cents.

This only gets us the number of coins, not the exact coins themselves. We can rectify this by adding a corresponding "last coin used" entry for each entry in the array. The "last coin used" entry for x will store which case in the recurrence "won" for that x. Now once we reach 100 cents, we can *backtrack* by repeatedly subtracting the value of the last coin used. The sequence of last coins used is the set of change that the machine should dispense. This technique is often useful in dynamic programming algorithms.

***Prompt:** An inversion is a pair of two elements in a permutation (the integers $$1 \ldots n$$ arranged in some order) which are "out of order" with respect to each other; that is, $$\tuple{x_j, x_k}\ \vert\ j < k \land x_j > x_k$$. For example, the permutation $$(2\ 1\ 4\ 5\ 3)$$ has three inversions: $$\tuple{2, 1}$$, $$\tuple{4, 3}$$, and $$\tuple{5, 3}$$. Given a permutation, determine how many inversions it contains.*

**Answer:** The most obvious solution is to try every possible pair of distinct elements using a nested loop, incrementing a counter each time you find an inversion. This would run in quadratic time. Interestingly, you can also perform a bubble sort and count the number of swaps you must perform– that number is exactly the number of inversions. Again, that's quadratic time.

The key insight is that this problem can be solved with the divide-and-conquer technique. In this technique you divide the problem into parts, solve each part recursively, and then do some work to combine the answers into an answer for the whole problem. In this case, break the permutation into halves. Assume that you can recusively obtain the number of inversions in the left half and in the right half separately, and assume that you have sorted both halves. For the recursion to work, the output you need to produce from these values is the number of inversions in the whole permutation combined, and the whole sorted permutation.

You can accomplish this easily by modifying merge sort, another divide-and-conquer algorithm. Merge sort works by breaking the input list into halves, recursively sorting each half, and then merging them by repeatedly taking the first element from one or the other sorted halves, depending on which element is smaller. We can use this exact algorithm for counting inversions, except that the merge operation needs to additionally maintain an inversion counter which is incremented by the remaining size of the left half any time an element from the right half is merged. Like mergesort, this has a $$\Theta(n\ lg\ n)$$ running time.

***Prompt:** Arbitrageurs make money by buying an asset and then immediately selling it in a different market to profit from a difference in price. One example of this is in the foreign exchange (forex) markets. Each currency pair (e.g. Euro / USD) has an exchange rate. If you can find a way to exchange your money between currency pairs such that you're able to return to your original currency with more money than you started with, you've found an arbitrage opportunity, which means risk-free profit! Given a list of currency pairs and their exchange rates, how would you determine whether there's an arbitrage opportunity?*

**Answer:** The structure of this problem is begging for it to be turned into a graph. You can represent currencies as nodes and traded currency pairs as edges, each with a weight corresponding to the exchange rate between its incident currencies. An arbitrage opportunity would then look like a cycle in the graph where the product of the edge weights is greater than one.

$$1 < rate_1 * rate_2 * \ldots * rate_n$$

It's not obvious how to find such a cycle efficiently. We might like to take advantage of the existing corpus of graph algorithms for cracking this problem, but generally paths through a weighted graph have a length which is the *sum* of the constituent edge weights. In this case, we're applying exchange rates multiplicatively so we want a path through the graph to have a length of the *product* of the constituent edge weights. 

We can translate the product into a sum by taking the log of both sides:

$$\lg 1 < \lg (rate_1 * rate_2 * \ldots * rate_n)\\
0 < \lg rate_1 + \lg rate_2 + \ldots + \lg rate_n$$

So now we've formulated an equivalent problem: finding a positive cycle in the graph where the edge weights are the log of the exchange rates. However, positive cycles aren't as easy to find as negative cycles. Multiply both sides by $$-1$$:

$$0 > -\lg rate_1 - \lg rate_2 - \ldots - \lg rate_n$$

Now the problem is to find a negative cycle in the graph where the edge weights are the negative log of the exchange rates. The [Bellman-Ford algorithm](https://en.wikipedia.org/wiki/Bellman%E2%80%93Ford_algorithm) can  find negative cycles in a graph.

The recurrence for Bellman-Ford defines $$B^{i}(x)$$ as the length of the shortest path from the starting node to $$x$$, using at most $$i$$ edges. This can be defined recursively:

$$B^0(\text{start}) = 0\\
B^0(x) = \infty \text{ for all } x\neq \text{start}\\
B^i(x) = \min\limits_{y \in V} \left\{B^{i-1}(y) + w_{yx}\right\}$$

Finding $$B^{\vert V \vert}(x)$$ for all nodes $$x$$ would take $$\Theta(\vert V \vert^3)$$ time if you build a table from the recurrence. Standard dynamic programming optimization techniques can bring this down. You can also use an adjacency list to only consider incident nodes in the $$\min$$, rather than all nodes. In the end, Bellman-Ford can be optimized down to $$\Theta(\vert V \vert \vert E \vert)$$:

```
repeat |V|-1 times:
	for each edge (j, k)
		B[k] <- min(B[k], B[j] + w[j, k])
```

Now that we have the shortest distance from our starting node (our starting currency) to every other node, we can check for negative cycles. If there's a negative cycle, then there's going to be no true shortest path to all nodes: for the nodes reachable from the negative cycle, you can go through the cycle as many times as you want to make the shortest distance as low as you want. Since we only iterated the B-F algorithm $$\vert V \vert-1$$ times, those very meandering shortest paths won't be accounted for. We can detect negative cycles by checking each edge and the "shortest" path lengths to its incident vertices:

```
for each edge (j, k)
	if (B[j] + w[j, k] < B[k])
		There is a negative cycle through (j, k)
```

Note: This is a bad interview question. Please don't ask it. It's so tricky in an interview setting that it basically only selects for people who've seen the problem before.

***Prompt:** How would you reduce lock contention in multithreaded code?*

**Answer:** One technique that often helps to reduce contention is to use separate reader-writer locks. You can also introduce finer-grained locking so that you obtain a lock on only part of your data structure from each thread. Finally, you might want to try lock-free programming techniques (e.g. a compare-and-swap loop).

***Prompt:** How do you deal with deadlock?*

**Answer:** A good practice for avoiding deadlock within an application is to impose a total ordering on locks and always acquire them in the same order in different places in your code. When potential deadlock is unavoidable, you can keep track of a wait graph and abort one of your processes/transactions if there's a cycle (but be careful of livelock). Two-phase locking: acquire all of the locks you need *before* mutating the protected resource so that you can easily roll back by releasing the locks if there's an issue acquiring all of the locks you need.

***Prompt:** What's the difference between [covariant and contravariant](https://en.wikipedia.org/wiki/Covariance_and_contravariance_%28computer_science%29) type parameters? When would you use each? What are your restrictions when designing a covariant collection class?*

**Answer:** When you define a generic interface, covariant type parameters appear as the types of method arguments. They say "I don't care what type you give me, as long as it extends `T` so that I can treat it as a `T`." These are useful when you're defining *consumers*, like `Comparator`s. Contravariant type parameters, on the other hand, appear as method return types and say: "I don't care what type you expect, as long as it's a superclass of `T`, because I'm going to give you a `T`." These are useful when you're defining *producers*, like `Iterator`s. [Effective Java](https://www.amazon.com/Effective-Java-2nd-Joshua-Bloch/dp/0321356683/?tag=electronicfro-20) puts it succinctly: "producers extend, consumers super" or "PECS."

A problem occurs when you want to *consume* an object of a covariant type, and then turn around and *produce* an object of the same type contravariantly. Languages like C# and Scala will actually prevent you from doing this at compile time, and with good reason. It turns out that allowing code like that to compile will break the type system. For example, in Java, this issue frequently confounds beginners:

```java
List<List<? extends Number>> a = null;
List<List<Integer>> b = null;
a = b; // Does NOT compile.

List<? extends List<? extends Number>> a = null;
List<List<Integer>> b = null;
a = b; // You have to do this instead.
```

Here's how the type system would break down if that were not the case:

```java
List<List<? extends Number>> a = null;
List<List<Integer>> b = new ArrayList<>();
a = b; // This won't compile, but assume it does.

a.add(Lists.<Double>newArrayList(2.718));
Integer c = b.get(0).get(0);
// Now c is an Integer containing 2.718
```

(That's not to say that you can't have both covariant and contravariant type parameters on the same class, or even on the same method. For example, a `Function` class with an `apply` method should have a covariant argument type and a contravariant return type, but – critically – they're not the same type.)

```scala
trait Function1[-T, +R] {
    def apply(arg1: T): R
}
```

Since you can't have the same type parameter in both a covariant argument position and a contravariant return position, it turns out to be impossible to safely implement a mutable collection with a variant type parameter. Indeed, the mutable collections in Scala all have invariant type parameters. The immutable collections, however, have covariant type parameters, because of the "one weird trick" that immutable collections have up their sleeve. Here's how Scala defines the immutable List interface:

```scala
package scala.collection.immutable
class List[+A] {
    def head: A
    def tail: List[A]
    def prepend[B >: A] (x: B): List[B]
}
```

The list *produces* values of type `A` through its `head` method, so `A` is a contravariant type parameter. However, when it *consumes* values through its `prepend` method, it can accept anything which is a superclass of A. This would appear to violate our rule, but it doesn't– because `prepend` returns a *whole new collection* of the new, less specific type. Immutable collections will always return a new instance rather than modifying state within themselves, which is the key to achieving both covariance and contravariance.