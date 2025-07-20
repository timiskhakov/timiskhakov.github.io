---
layout: post
title: Jumping Through Loops
excerpt: Symmetric difference in four, three, and two iterations
tags: [go, data structures, algorithms]
---

Lately, I've been working on a task that involves calculating the difference between two slices, returning unique elements that are not shared between them. The description might sound a bit clumsy, let's look at an example. If we have two input slices `[1, 1, 2, 3, 3]` and `[3, 3, 4, 4]`, the result should be `[1, 2]` and `[4]`. All common elements, in this case `3`, should be excluded from both results, as well as any duplicates. Although the slices in the example are ordered, that isn't guaranteed. We can have an input like this: `[9, 5, 7]` and `[3, 9, 1]`, and the result should be `[5, 7]` and `[3, 1]`. The order of elements in the result doesn't matter either, for instance, `[7, 5]` and `[1, 3]` would also be a valid result. The key requirement is that each resulting slice contains only unique elements not present in the other slice.

If you're familiar with set theory, this problem might resemble finding the [symmetric difference](https://en.wikipedia.org/wiki/Symmetric_difference), or the disjunctive union, of two sets. However, in this case, we're working with slices, not sets, that may contain duplicates, and instead of producing a single set, we return two separate slices, each containing the unique elements not found in the other. For the lack of a better term, however, I'll continue referring to the problem as finding the symmetric difference.

At first glance, this might seem like a simple problem, and we will indeed begin with a simple solution. However, in the spirit of this blog, we'll explore interesting optimizations of the initial approach. As mentioned, we'll start with a simple straightforward solution, then make a relatively obvious optimization, and finally revisit the data structure we explored the last time to see if it can help us out this time (spoiler: it totally can). But let's start with the basics.

# The Basics

The first idea that probably comes to mind is simply to find the set difference for each slice and call it a day:

```go
func diff[T comparable](first, second []T) []T {
  seen := make(map[T]bool, len(first)+len(second))
  for _, e := range second {
    seen[e] = true
  }
  result := make([]T, 0, len(first))
  for _, item := range first {
    if !seen[item] {
      result = append(result, item)
      seen[item] = true
    }
  }
  return result
}
```

First, we make the function a bit generic, but still require `T` to be `comparable` as we need to use it as `map`'s key. Then we go over all items from the `second` slice and mark them as seen. Finally, we range over the `first` slice, populating the `result` slice with items not marked yet, and only then marking them as seen.

Now, to calculate the symmetric difference, we need to `diff` both inputs in different order:

```go
func BasicSymmDiff[T comparable](first, second []T) ([]T, []T) {
  return diff(first, second), diff(second, first)
}
```

Looks easy, right? But if we take a closer look, we'll notice that we need to iterate over each slice twice: once in each `diff` call. That's not necessarily bad, but we can definitely do better.

# Doing Better

In `diff`, we are using `map[T]bool` for storing elements found in both slices. This is common practice in Go for making sets of `T`, but we can also use this `map` so that it can tell whether an element exists in one of the slices or in both. With that change we can remove one loop over a slice:

```go
func BetterSymmDiff[T comparable](first, second []T) ([]T, []T) {
  seen := make(map[T]bool, len(first)+len(second)) // (1)
  for _, e := range first {
    seen[e] = false
  }
  secondUnique := make([]T, 0, len(second)) // (2)
  for _, e := range second {
    if _, ok := seen[e]; ok {
      seen[e] = true
    } else {
      seen[e] = false
      secondUnique = append(secondUnique, e)
    }
  }
  firstUnique := make([]T, 0, len(first)) // (3)
  for _, e := range first {
    if !seen[e] {
      seen[e] = true
      firstUnique = append(firstUnique, e)
    }
  }
  return firstUnique, secondUnique
}
```

There's more things happening in the function now, so let's break it down:

1. First, we create the same `seen` map which will hold information on whether we have marked an element or not. However, this time round, we go over the `first` slice and mark its items as `false`. That indicates that the item only exists in the `first` slice.
2. Next, we range over the `second` slice, but here we check if we have the element in `seen` or not. If we do, we set it to `true`, meaning both slices have it. But if not, we add the element there marked with `false` and append it to the second resulting slice.
3. Finally, we come back to the `first` slice and check its items against `seen` again. Now we are interested in only `false` elements, which means that they are present only in `first` or `second`. So we add them to the first resulting slice and mark as `true`.

Now with only three iterations over the input slices we get the same result. It's especially beneficial if the `second` slice, which we iterate only once, is larger than the `first` one.

Let's see if we get any benefits out of this optimization. For the benchmark, I use two equally sized slices of strings, each with 6 characters. This is close to the conditions of the task I mentioned at the beginning of the post, but we will run it with different numbers of strings in each slice:

```
goos: darwin
goarch: arm64
pkg: github.com/timiskhakov/symmdiff
cpu: Apple M2 Pro
    1_000  BenchmarkBasicSymmDiff-12   11172     104939 ns/op     251204 B/op     20 allocs/op
    1_000  BenchmarkBetterSymmDiff-12  14503      82811 ns/op     141986 B/op     11 allocs/op
   10_000  BenchmarkBasicSymmDiff-12     811    1323188 ns/op    2075239 B/op    134 allocs/op
   10_000  BenchmarkBetterSymmDiff-12   1074    1106430 ns/op    1201434 B/op     68 allocs/op
  100_000  BenchmarkBasicSymmDiff-12      79   14494739 ns/op   17192635 B/op   1032 allocs/op
  100_000  BenchmarkBetterSymmDiff-12    100   11042279 ns/op   10202004 B/op    517 allocs/op
1_000_000  BenchmarkBasicSymmDiff-12       3  343928236 ns/op  255699381 B/op  16403 allocs/op
1_000_000  BenchmarkBetterSymmDiff-12      4  287940906 ns/op  143851608 B/op   8198 allocs/op
```

Nice, we get some benefits in both time and memory. Honestly, that's good enough for me, and this is the solution I chose for the original problem. However, if you read this blog before, you probably know the drill by now: always raise the bar! So, can we do just two iterations, one per input slice? Sounds a bit challenging, but with a little help from an old friend we can make it happen.

# An Old Friend

Remember [sparse sets](sparse-sets-and-where-to-find-them)? We have explored them a while ago and found an interesting usage in building custom dictionaries in C# for solving and optimizing a specific problem. The core idea from sparse sets can also help us here with finding the symmetric difference. We already had a good look into how they work under the hood, so I won't be diving into that now. Unlike in the post about sparse sets, though, this time around we will actually use sets, not dictionaries or maps.

But first, let's set the stage defining the set as a `struct` and add a constructor:

```go
type sparseSet[T comparable] struct {
  sparse map[T]int
  dense  []T
}

func newSparseSet[T comparable](size int) *sparseSet[T] {
  return &sparseSet[T]{
    sparse: make(map[T]int, size),
    dense:  make([]T, 0, size),
  }
}
```

We would need `sparse` as `map[T]int` to store the set's elements, and we'll also need to store them in a slice called `dense`. If you've read the post on sparse sets linked above, you may be wondering why we're not using a slice for `sparse`, and that's because our items have a `comparable` type. We're likely to sacrifice some performance due to hashing and collision resolution in a `map`, but the trade-off will be worth it. To be honest, we can't really call it `sparse` as there's nothing sparse in a `map`. But again, for the lack of a better name, let's call it that anyway[^1]. Nonetheless, `sparse` still holds one of its original properties: `int` values serve as indexes of the element in `dense`. In that sense, `sparse` values still point to `dense`, but that doesn't work the other way around.

Next stop, we need to define a method to check if an item is in the set, as well as methods to add elements to and remove them from the set:

```go
func (s *sparseSet[T]) has(item T) bool {
  _, ok := s.sparse[item]
  return ok
}

func (s *sparseSet[T]) add(item T) {
  if !s.has(item) {
    s.sparse[item] = len(s.dense)
    s.dense = append(s.dense, item)
  }
}

func (s *sparseSet[T]) remove(item T) {
  if index, ok := s.sparse[item]; ok {
    last := s.dense[len(s.dense)-1]
    s.sparse[last] = index
    s.dense[index] = last
    delete(s.sparse, item)
    s.dense = s.dense[:len(s.dense)-1]
  }
}
```

Let's start with `has`, since it's the most simple one. In order to check if an `item` belongs to the set, we just need to see if it's in `sparse` or not, and that's it.

Continuing with `add`, we first check whether the `item` has already been added. If not, we insert it into `sparse` with the current length of `dense` as a value, at the same time appending it to `dense`. This way, the value in `sparse` points to the `item`'s position in `dense`.

To remove an `item`, we need to perform a few steps. First, check if it exists in `sparse`. If it does, retrieve the last element from `dense` and move it to the position of the `item` being removed, both in `dense` and by updating its index in `sparse`. Finally, remove the `item` from `sparse` and shorten `dense` by one.

Since we've been tracking the length of `dense` as elements are added or removed, we can just return the slice once we need it. Yes, I know, technically returning `dense` directly does leak `sparseSet`'s abstraction. However, that should be fine for our purposes: `sparseSet` is an internal struct, that will be used only within the scope of a single client call.

Now we can see how `sparseSet` and its leaky abstraction help solve the initial problem cutting the number of iterations down to two.

# Leaky Abstraction's Help

If we had another close look, this time at `BetterSymmDiff`, we would see that we don't really need the second iteration over the `first` slice. We just need a way to remove items from the `seen` map and get its keys, that have `false` values, as a slice. This is exactly what the `sparseSet` will do for us.

I'm running out of meaningful names for implementations, so... `SparseSymmDiff`?

```go
func SparseSymmDiff[T comparable](first, second []T) ([]T, []T) {
  firstUnique := newSparseSet[T](len(first)) // (1)
  for _, e := range first {
    firstUnique.add(e)
  }
  seen := make(map[T]bool, len(second)) // (2)
  secondUnique := make([]T, 0, len(second))
  for _, e := range second { // (3)
    if seen[e] {
      continue
    }
    if firstUnique.has(e) {
      firstUnique.remove(e)
    } else {
      secondUnique = append(secondUnique, e)
    }
    seen[e] = true
  }
  return firstUnique.dense, secondUnique // (4)
}
```

Again, we have quite many things happening in the function, so let's go over it step by step:
1. We create the `sparseSet` which will hold unique items that only exist in the `first` slice. So we immediately range over this slice and add its items to the set. That's the first iteration.
2. Then we create `seen` and `secondUnique`. Both are used only for storing elements from the `second` slice.
3. We range over the `second` slice in order to do the following:
	- We check if we have seen an element and added it to `seen` already. If that's the case, we skip it.
	- Otherwise, we check if `firstUnique` has this element. If it does, we remove it from there as it's an element that both slices share. If not, the element is unique to the `second` slice, so we add it to `secondUnique`.
	- Finally, we mark the element as `seen`.
4. That was the second iteration. Since all defined operations on `sparseSet` have constant time, we don't need to do more loops. As we've discussed in the `sparseSet` implementation, we only need to access its underlying `dense` in order to get the resulting slice.

Let's see how this implementation performs in our benchmark:

```
    1_000  BenchmarkBasicSymmDiff-12   11355     103508 ns/op     251205 B/op     20 allocs/op
    1_000  BenchmarkBetterSymmDiff-12  14708      81521 ns/op     141987 B/op     11 allocs/op
    1_000  BenchmarkSparseSymmDiff-12  17520      68372 ns/op     142034 B/op     13 allocs/op
   10_000  BenchmarkBasicSymmDiff-12     840    1265496 ns/op    2075239 B/op    134 allocs/op
   10_000  BenchmarkBetterSymmDiff-12   1150    1026802 ns/op    1201437 B/op     68 allocs/op
   10_000  BenchmarkSparseSymmDiff-12   1368     871613 ns/op    1201490 B/op     70 allocs/op
  100_000  BenchmarkBasicSymmDiff-12      87   13369259 ns/op   17192594 B/op   1032 allocs/op
  100_000  BenchmarkBetterSymmDiff-12    100   10976453 ns/op   10201398 B/op    517 allocs/op
  100_000  BenchmarkSparseSymmDiff-12    124    9538148 ns/op   10201440 B/op    519 allocs/op
1_000_000  BenchmarkBasicSymmDiff-12       4  324443198 ns/op  255696760 B/op  16401 allocs/op
1_000_000  BenchmarkBetterSymmDiff-12      4  285507625 ns/op  143851608 B/op   8198 allocs/op
1_000_000  BenchmarkSparseSymmDiff-12      5  220074200 ns/op  143855769 B/op   8200 allocs/op
```

Sweet, the results stay consistent regardless of slice sizes. However, the benchmark we use is very specific to the task I was solving, with the inputs being strings of size 6 characters. But what if we try to run more benchmarks?

# More Benchmarks

Up until now, we have been benchmarking using slices with the same amount of strings, each having the same length. But what if we supply different values? Say, 1,000 items of size 6 in the first slice, and 10,000 items of size 16 in the second?

```
BenchmarkBasicSymmDiff-12           1844      646312 ns/op   1053985 B/op       68 allocs/op
BenchmarkBetterSymmDiff-12          2901      408493 ns/op    617101 B/op       35 allocs/op
BenchmarkSparseSymmDiff-12          2397      479184 ns/op    671757 B/op       41 allocs/op
```

The result of course is in favor of the "better" implementation as it's only iterating over the bigger second slice once. Which seemingly beats having to deal with the overhead provided by the sparse implementation. 

We can also try slices of integers: 100,000 integers each being as big as `10000`:

```
BenchmarkBasicSymmDiff-12            312     3653052 ns/op  11064022 B/op     1032 allocs/op
BenchmarkBetterSymmDiff-12           399     2949215 ns/op   6334801 B/op      517 allocs/op
BenchmarkSparseSymmDiff-12           513     2307584 ns/op   6334854 B/op      519 allocs/op
```

No surprises here. But what if we increase a possible maximum value up to a million?

```
BenchmarkBasicSymmDiff-12            192     6092303 ns/op  11064029 B/op     1032 allocs/op
BenchmarkBetterSymmDiff-12           256     4668118 ns/op   6334799 B/op      517 allocs/op
BenchmarkSparseSymmDiff-12           231     5142987 ns/op   6334852 B/op      519 allocs/op
```

Again, the "better" one is doing better in this benchmark. Which can be explained by the fact that `SparseSymmDiff` is heavy on maps, and with larger integers as keys, hashing and bucket lookups tend to become slower due to less efficient hash distribution and potential cache impacts.

As we can see, benchmarking functions that come with trade-offs is tricky business. In certain scenarios, we can gain benefits by trading memory over the number of loop iterations, but in others we can't. That's why it's important to know what kind of a problem and data we are dealing with, do our own benchmarks, and trade off accordingly.

You can check out the code from this post on GitHub: [symmdiff](https://github.com/timiskhakov/symmdiff).

# Footnotes

[^1]: By now, you've probably noticed the running theme of this post: finding the symmetric difference that isn't quite a symmetric difference, and using a sparse data structure that isn't really sparse.