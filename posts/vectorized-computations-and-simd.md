# Vectorized Computations and SIMD

Recently I've been searching for some ways to improve performance for my never-to-be-released roguelike game that I make in spare time purely for the sake of programming and my own entertainment. I stumbled upon exploiting vectorized computations and SIMD usage.

But before diving into the topic let's try to come up with the simplest problem we could apply this solution to...

## Problem

...which is computing a sum of a large array of integers. The larger the better. Yeah, I know, the problem itself doesn't sound _that_ exciting, but bare with me, you might be surprised at the end. Also, for the demo I try to keep things as simple as possible. Besides, just making things fast is fun, isn't it?

## Naïve Solution

How would we approach the problem? Perhaps, you already wrote some code in your mind. Chances are it looks quite similar to this one:
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
(As we know when dealing with arrays LINQ might not be a very good tool as it brings the whole `IEnumerable` infrastructure along exposing the enumerator and checking for the next available element.)

Writing production-ready code we usually stop here as this solution is good enough in almost all cases. But sometimes we really need to gain speed preferably without a plunge into a swirl of unmanaged code. In such cases let me bring vectorized computations on the table.

## Vectorization

Vectorization in general is a term for applying operations to a set of elements rather than a single scalar element. That is, instead of doing:
```
8 + 13 = 21
76 + 19 = 95
```
We do:
```
(8, 76) + (13, 19) = (21, 95)
```
Combining elements into a vector and replacing two operations with a single one.

Modern processors can do the same, it's called SIMD, which stands for single instruction, multiple data. SSE (Streaming SIMD Extensions) that implements this feature is an instruction set extension that was introduced by Intel in their Pentium 3 processor in 1999. It allowed operations on vectors containing 64 bits of data producing the same size result. That is, we could fit 2 integer values, each had a size of 32 bits, into a vector.

Over time in the following processor models the instruction set evolved into SSE2, SSE3 and so on. Nowadays modern mass market CPUs allow operate on vectors that have a size of 256 bits (at least I have that one on my machine). That means we can pack 8 `int`s, or 16 `byte`s, or 4 `double`s into a single vector.

The approach of doing parallel computations on a single core is not new for C or C++ programming, but in our .NET world it has appeared since .NET Framework 4.6 in 2015.

But let's go back to our code. We can reach the SIMD-enabled types, like, `Vector2`, `Vector3`, `Vector4`, `Vector<T>`, through `System.Numerics` namespace. Even if our hardware doesn't support SIMD instructions, the types provide a software fallback.

Before we go anywhere, let's explore how vectors work under the hood by writing a super simple example in which we add together 2 arrays containing 8 integers:
```csharp
var array1 = new[] {1, 2, 3, 4, 5, 6, 7, 8};
var array2 = new[] {1, 2, 3, 4, 5, 6, 7, 8};

// The way we do it normally
var result1 = new int[array1.Length];
for (var i = 0; i < result1.Length; i++)
{
	result1[i] = array1[i] + array2[i];
}

// Trying to do the same thing, but with vectors
var result2 = new int[array1.Length];
(new Vector<int>(array1) + new Vector<int>(array2)).CopyTo(result2);
```
If we open the assembly code produced by the first part, we can see that we repeat in a loop the following instructions 8 times:
```assembly
mov     r9d, dword ptr [...]    ; obtaining a value from array1
mov     r10d, dword ptr [...]   ; obtaining a value from array2
add     r9d, r10d               ; perform the addition operation on two values
mov     dword, ptr [...]        ; save the result to result1
```
While for the second part we only have:
```assembly
vmovupd ymm0, ymmword ptr       ; load array1 into the first vector
vmovupd ymm1, ymmword ptr       ; load array2 into the second vector
vpaddd  ymm0, ymm0, ymm1        ; perform the addition operation on two vectors
vmovupd ymmword ptr [...], ymm0 ; copy the vector to the destination array
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
2. We go through the our array vector by vector, adding each vector to the result.
3. Then we summarize all the elements that reside in the vector result into a scalar integer result.
4. Since our initial array can have a length not divided evenly by the vector size, we have to deal with the remaining elements. We just add each element to the final result.

Let's compare two methods that we wrote so far. As always I use [`BenchmarkDotNet`](https://github.com/dotnet/BenchmarkDotNet):
```
|   Method |        Mean |     Error |    StdDev |      Median |
|--------- |------------:|----------:|----------:|------------:|
| NaiveSum |   474.61 us |  3.327 us |  2.949 us |   473.74 us |
|  SimdSum |    79.79 us |  2.317 us |  6.795 us |    76.83 us |
```
The spec I run this benchmark under:
- macOS 10.15.2
- Intel Core i7-8569U CPU 2.80GHz (Coffee Lake)
- .NET Core SDK=3.1.100

Bare in mind that we compare two methods one of which is CPU-dependent, so, your results might be very different.

## Conclusion

We might use a really simple problem as an example to explore vectorized computation, but even in this case the final result is quite surprising. Of course, not all algorithms can use vectorization to speed things up. But even if you use one that can, please always measure your performance before doing any tweaks.

You can check out this post's code on GitHub: [VectorizedComputationsAndSimd](https://github.com/timiskhakov/VectorizedComputationsAndSimd).

## Further Reading

- [Parallelism on a Single Core - SIMD with C#](https://instil.co/2016/03/21/parallelism-on-a-single-core-simd-with-c/)
- [SIMD in Depth - Performance and Cost in C# and C++](https://instil.co/2016/04/07/simd-performance-with-csharp-and-cpp/)
- [Using .NET Hardware Intrinsics API to accelerate machine learning scenarios](https://devblogs.microsoft.com/dotnet/using-net-hardware-intrinsics-api-to-accelerate-machine-learning-scenarios/)
- [Hardware Intrinsics in .NET Core](https://devblogs.microsoft.com/dotnet/hardware-intrinsics-in-net-core/)
- [Sasha Goldshtein — The Vector in Your CPU: Exploiting SIMD for Superscalar Performance](https://www.youtube.com/watch?v=WeJ8b3WRSmM)
