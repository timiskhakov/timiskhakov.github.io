# Vectorized Computations and SIMD

Recently I've been searching for ways to improve the performance of my never-to-be-released roguelike game. I make it in my spare time purely for the sake of programming and my own entertainment, so I often diverge into performance-related bikeshedding. This time I stumbled upon vectorized computations and SIMD to speed up some array-based algorithms in the game.

Before diving into the topic let's try to come up with the simplest problem we could apply this solution to.

## Problem

For the sake of demo I try to keep things as simple as possible, so let's compute a sum of a large array of integers, the larger the better. Yeah, I know, the problem itself doesn't sound that exciting but bear with me, you might be surprised in the end. Besides, making fast things even faster is always fun, isn’t it?

## Naïve Solution

How would we approach the problem? Perhaps you already wrote some code in your mind. Chances are it looks quite similar to this:
```csharp
public static int NaiveSum(int[] array)
{
  var result = 0;
  for (var i = 0; i < array.Length; i++)
  {
    result += array[i];
  }

  return result;
}
```
(Another option would be using LINQ by writing `array.Sum()`, but as we know when dealing with arrays LINQ might not be a very good tool as it brings the whole `IEnumerable` infrastructure along. In this case, exposing the enumerator and checking for the next available element make things slow.)

Writing production-ready code, we usually stop here as this solution is just good enough in nearly all cases. However, sometimes we really need to gain some speed preferably without plunging into a swirl of unmanaged code. Let me introduce vectorized computations.

## Vectorization

Vectorization, in general, is a term for applying operations to a set of elements rather than to a single scalar element. That is, instead of doing:
```
8 + 13 = 21
76 + 19 = 95
```
We do:
```
(8, 76) + (13, 19) = (21, 95)
```
Combining elements into a vector and replacing two operations with one.

Modern processors can do a quite similar thing that's called SIMD, which stands for single instruction, multiple data. SSE (Streaming SIMD Extensions) that implements this feature is an instruction set extension that was introduced by Intel in their Pentium 3 processor in 1999. It allowed programmers to operate on vectors containing 64 bits of data producing a result of the same size. That is, we could fit two integer values, each having a size of 32 bits, into a single vector.

Over time every new processor model brought along a more powerful extension: SSE2, SSE3 and so on. Nowadays modern CPUs have AVX (or Advanced Vector Extensions) allowing to operate on vectors that have a size of 256 bits.

The approach of doing parallel computations on a single core is not new for other programming languages, but in our .NET world it has appeared starting from .NET Framework 4.6 in 2015.

We can reach the SIMD-enabled types through `System.Numerics` namespace. Even if our hardware doesn't support SIMD instructions, the types provide a software fallback. The type we are most interested in is `Vector<T>`. Its size depends on `T`. Since we have only 256 bits at our disposal we can pack 16 `byte`s, or 8 `int`s, or 4 `double`s into a single vector.

Before we go any further with our example, let's explore how vectors work under the hood. We will write a simple example in which we add together two arrays containing 8 integers:
```csharp
var array1 = new[] {1, 2, 3, 4, 5, 6, 7, 8};
var array2 = new[] {1, 2, 3, 4, 5, 6, 7, 8};

// Part 1: the way we do it normally
var result1 = new int[array1.Length];
for (var i = 0; i < result1.Length; i++)
{
  result1[i] = array1[i] + array2[i];
}

// Part 2: trying to do the same thing, but with vectors
var result2 = new int[array1.Length];
(new Vector<int>(array1) + new Vector<int>(array2)).CopyTo(result2);
```
If we open the assembly code produced by the first part, we can see that we loop over the following instructions 8 times:
```assembly
mov     r9d, dword ptr [...]    ; obtaining a value from array1
mov     r10d, dword ptr [...]   ; obtaining a value from array2
add     r9d, r10d               ; perform addition on two values
mov     dword, ptr [...]        ; save the result to result1
```
While for the second part we only have:
```assembly
vmovupd ymm0, ymmword ptr       ; load array1 into the first vector
vmovupd ymm1, ymmword ptr       ; load array2 into the second vector
vpaddd  ymm0, ymm0, ymm1        ; perform addition on two vectors
vmovupd ymmword ptr [...], ymm0 ; copy the vector to the array
```
So, we just replaced a loop that repeats a bunch of instructions 8 times with just a few instructions. Hence, the name "single instruction, multiple data" and also that's why sometimes people call vectorized computations "parallelization on a single processor".

## Vector Solution

Coming back to our problem with the knowledge we obtained, let's write a new method that uses vectors:
```csharp
public static int SimdSum(int[] array)
{
  var vector = Vector<int>.Zero;
  var i = 0;
  for (; i <= array.Length - Vector<int>.Count; i += Vector<int>.Count)
  {
    vector += new Vector<int>(array, i);
  }

  var result = 0;
  for (var j = 0; j < Vector<int>.Count; j++)
  {
    result += vector[j];
  }

  for (; i < array.Length; i++)
  {
    result += array[i];
  }

  return result;
}
```
Here we do a couple of things to split processing into vectors:
1. We create a vectorized result containing all zeros. Since it's a vector of integers, its size is 8.
2. We go through our input array vector by vector, adding each vector to the result.
3. Then we summarize all the elements residing in the vector result into a scalar integer result.
4. Since our input array can have a length not divided evenly by the vector size, we have to deal with the remaining elements. We just add each element to the final result.

Let's compare two methods that we wrote so far trying to find a sum of an array of 1 million integers each having a value from 1 to 1000 (as always I use [`BenchmarkDotNet`](https://github.com/dotnet/BenchmarkDotNet)):
```
|   Method |        Mean |     Error |    StdDev |      Median |
|--------- |------------:|----------:|----------:|------------:|
| NaiveSum |   474.61 us |  3.327 us |  2.949 us |   473.74 us |
|  SimdSum |    79.79 us |  2.317 us |  6.795 us |    76.83 us |
```
The spec I ran this benchmark under:
- macOS 10.15.2
- Intel Core i7-8569U CPU 2.80GHz (Coffee Lake)
- .NET Core SDK=3.1.100

Bear in mind, though, we compare two methods, one of which is CPU-dependent, so your result might be very different.

## Conclusion

We might use a really simple problem as an example to explore vectorized computation, but even in this case, the final result is quite surprising. Of course, not all algorithms can use vectorization to speed things up. But even if you use the one that can, please always measure your performance before doing any tweaks.

You can check out code from this post on GitHub: [VectorizedComputationsAndSimd](https://github.com/timiskhakov/VectorizedComputationsAndSimd).

## Further Reading

- [Parallelism on a Single Core - SIMD with C#](https://instil.co/2016/03/21/parallelism-on-a-single-core-simd-with-c/)
- [SIMD in Depth - Performance and Cost in C# and C++](https://instil.co/2016/04/07/simd-performance-with-csharp-and-cpp/)
- [Hardware Intrinsics in .NET Core](https://devblogs.microsoft.com/dotnet/hardware-intrinsics-in-net-core/)
- [Using .NET Hardware Intrinsics API to accelerate machine learning scenarios](https://devblogs.microsoft.com/dotnet/using-net-hardware-intrinsics-api-to-accelerate-machine-learning-scenarios/)
- [Sasha Goldshtein — The Vector in Your CPU: Exploiting SIMD for Superscalar Performance](https://www.youtube.com/watch?v=WeJ8b3WRSmM)
