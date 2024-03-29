---
layout: post
title: Haystacks, Needles, and Hardware Intrinsics
excerpt: .NET Core 3.0 allows us to dive a bit deeper into SIMD and intrinsics
tags: [c#, simd, algorithms]
---

In [the previous post](vectorized-computations-and-simd), we have explored SIMD-enabled types, such as vectors, provided by the `System.Numerics` namespace. They allow us to vectorize some array-based algorithms to speed up performance. However, since we were running code on .NET Core 3.1, we kinda ignored the elephant in the room — hardware intrinsics. Hardware intrinsics are special functions that are converted into CPU-specific instructions. They provide us with the same functionality as vectors from `System.Numerics` giving more flexibility and exposing additional instructions we can leverage on. Starting from .NET Core 3.0 hardware intrinsics are available under the `System.Runtime.Intrinsics` namespace.

Remember this example from the previous post?

```csharp
var result = new int[8];

var vectorA = new Vector<int>(new[] {1, 2, 3, 4, 5, 6, 7, 8});
var vectorB = new Vector<int>(new[] {1, 2, 3, 4, 5, 6, 7, 8});
vectorA += vectorB;
vectorA.CopyTo(result);
```

It was translated into the following assembly code:

```asm
vmovdqu ymm0, ymmword ptr       ; load first array into vector A
vmovdqu ymm1, ymmword ptr       ; load second array into vector B
vpaddd  ymm0, ymm0, ymm1        ; perform addition on two vectors
vmovdqu ymmword ptr [...], ymm0 ; copy the vector to the array
```

Using intrinsics we can write same code like this:

```csharp
var result = new int[8];

fixed (int* pArray1 = new[] {1, 2, 3, 4, 5, 6, 7, 8})
fixed (int* pArray2 = new[] {1, 2, 3, 4, 5, 6, 7, 8})
fixed (int* pResult = result)
{
  var vectorA = Avx.LoadVector256(pArray1);
  var vectorB = Avx.LoadVector256(pArray2);
  vectorA = Avx2.Add(vectorA, vectorB);
  Avx.Store(pResult, vectorA);
}
```

First two `Avx.LoadVector256` are converted into the `_mm256_loadu_si256` intrinsic which, according to the [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/), translates to the `vmovdqu` instruction. Then `Avx2.Add` converts into `_mm256_add_epi32` that adds vectors together via `vpaddd`. Finally, we store the result into the memory using the `_mm256_storeu_si256` intrinsic that gets translated to `vmovdqu`. As you might have noticed, here we got the same instructions as for the example above.

One of the benefits of using intrinsics is the ability to choose which instruction set to use. For instance, here we are exploiting AVX2 because we perform addition on 256-bits of integer data with 8 elements per vector, although, nothing stops us from choosing a different instruction set if we want to use smaller vectors, for example:

```csharp
fixed (int* pArray = new[] {1, 2, 3, 4})
{
  var vector = Sse2.LoadVector128(pArray); // => 4 elements
}
```

As you can see, a downside might be the necessity of pinning the array in the memory to obtain the pointer for loading and saving vectors.

Another drawback is that unlike vectors intrinsics don't have an abstraction that provides a software fallback if the hardware doesn't support a particular instruction set. We have to be careful with that, otherwise, we get an exception that the set is not supported on the machine. We can use the `IsSupported` property to check if it's available:

```csharp
if (Avx2.IsSupported)
{
  // Do things
}
```

Let’s explore what intrinsics are capable of.

# Problem

Last time we calculated a sum of an array of integers. Let's do something different this time. Say, we have a needle string and we need to find a position of its first occurrence in the haystack string; if it's not found, we return `-1`. You guessed it right, we are about to implement our own `IndexOf()`.

# Naive Solution

We will start with the [naive string search algorithm](https://en.wikipedia.org/wiki/String-searching_algorithm#Na%C3%AFve_string_search) as a baseline:

```csharp
public static int NaiveIndexOf(string haystack, string needle)
{
  for (var i = 0; i <= haystack.Length - needle.Length; i++)
  {
    var j = 0;
    for (; j < needle.Length; j++)
    {
      if (haystack[i + j] != needle[j]) break;
    }

    if (j == needle.Length)
    {
      return i;
    }
  }

  return -1;
}
```

It's not very efficient, but it does its job and might be a good starting point for us.

# Built-in Methods

We already mentioned one built-in method for solving the problem, `String.IndexOf()`. We could also use `Regex` to write something like:

```csharp
public static int RegexIndexOf(string haystack, string needle)
{
  var match = Regex.Match(haystack, needle, RegexOptions.Compiled);
  return match.Success ? match.Index : -1;
}
```

We will do benchmarking to check which method is faster at the end of the post. In the meantime let's write our own SIMD-friendly solution.

# SIMD Algorithm

For the SIMD solution, we will use an algorithm described in Wojciech Muła's [SIMD-friendly algorithms for substring searching](http://0x80.pl/articles/simd-strfind.html#algorithm-1-generic-simd). It's based on the naive string search algorithm that's modified for SIMD usage.

1. We load the first and last characters of the needle into `needleFirst` and `needleLast` vectors.
2. We go through the haystack block by block. A block has the same size as a vector. With each iteration, we’re loading data into two vectors: the first one starts with the beginning of the block, the second one starts with the offset of the needle length minus one. We call them `blockFirst` and `blockLast`.
3. Next, we compare `needleFirst` with `blockFirst`, and `needleLast` with `blockLast`, storing results into `matchFirst` and `matchLast` respectively.
4. Then we perform the bitwise AND operation and save it to the `and` vector.
5. Finally, we compute the mask of `and`. If the mask is equal to `0`, there is no match found, we carry on processing the next block. Otherwise, we have potential candidates. For each mask's bit set to `1`, we extract its position, calculate the first and last indices of the candidate within the haystack, and perform candidate's character by character comparison against the needle.

It might sound complex, but let's try it out on an example and see that it's actually a quite simple approach. Say, we have this string as a haystack: `The cake is a lie`. The needle we are looking for is, well, `cake`.

Just to keep things simple, here we will use 8 elements vectors. We start with defining `needleFirst`, `needleLast`, `blockFirst`, and `blockLast`:

```
needleFirst = ['c', 'c', 'c', 'c', 'c', 'c', 'c', 'c']
needleLast  = ['e', 'e', 'e', 'e', 'e', 'e', 'e', 'e']
blockFirst  = ['t', 'h', 'e', ' ', 'c', 'a', 'k', 'e']
blockLast   = [' ', 'c', 'a', 'k', 'e', ' ', 'i', 's']
```

According to step 3, we compare `needleFirst` with `blockFirst`, and `needleLast` with `blockLast`:

```
matchFirst  = [0, 0, 0, 0, 1, 0, 0, 0]
matchLast   = [0, 0, 0, 0, 1, 0, 0, 0]
```

Then we calculate `and` between the two and compute the mask:

```
and         = [0, 0, 0, 0, 1, 0, 0, 0]
mask        = 16
```

What value 16 tells us is that if we were to imagine our 8 elements vector in the form of a binary number it would look like `00001000`. (Keep in mind, though, it's a [little-endian representation](https://en.wikipedia.org/wiki/Endianness#Little-endian), so for us, humans, a more readable format would be big-endian: `00010000`, or just `10000`). That's exactly number 16 in the decimal numeral system.

Finally, we obtain a zero-based position of the first bit set to `1` which is 4 in the little-endian representation, and calculate candidate's first and last indices: 4 and 7. We check our candidate: `cake`, which happens to be the needle.

# SIMD Implementation

Following Wojciech Muła's blog post, we can find an [implementation](http://0x80.pl/articles/simd-strfind.html#sse-avx2) written in C++ that uses intrinsics. Let's port it to C#:

```csharp
public static unsafe int IntrinsicsIndexOf(string haystack, string needle)
{
  fixed (char* pHaystack = haystack)
  fixed (char* pNeedle = needle)
  {
    Vector256<ushort> needleFirst = Vector256.Create(pNeedle[0]);
    Vector256<ushort> needleLast = Vector256.Create(pNeedle[needle.Length - 1]);
    for (var i = 0; i < haystack.Length; i += Vector256<ushort>.Count)
    {
      var blockFirst = Avx.LoadVector256((ushort*) pHaystack + i);
      var blockLast = Avx.LoadVector256((ushort*) pHaystack + i + needle.Length - 1);

      var matchFirst = Avx2.CompareEqual(needleFirst, blockFirst);
      var matchLast = Avx2.CompareEqual(needleLast, blockLast);

      var and = Avx2.And(matchFirst, matchLast);
      var maskBytes = Avx2.MoveMask(and.AsByte());
      var mask = RemoveOddBits(maskBytes);
      while (mask > 0)
      {
        var position = GetFirstBit(mask);
        if (Compare(pHaystack, i + position + 1, pNeedle, 1, needle.Length - 2))
        {
          return i + position;
        }

        mask = ClearFirstBit(mask);
      }
    }
  }

  return -1;
}
```

We pin both strings, the haystack and the needle, in the memory to obtain their pointers. According to the algorithm we create two vectors `needleFirst` and `needleLast` that contain the needle's first and last characters respectively. Please note that we get `Vector256<ushort>` out of the `Create` method, not `Vector256<char>`, because vectors can only store integral and floating-point numeric types. Unlike the example above we use AVX vectors containing 256-bits of data, or 16 `ushort`s.

We go through the haystack vector by vector. With each iteration, we load haystack data into `blockFirst` and `blockLast`, trying to find a match and computing the mask. Unlike the C++ implementation, though, we have to do a few tricks with the mask in C#.

First, instrinsics don't have a function for computing the mask of the vector containing 16-bit values, like `ushort`. We have to represent it as a vector containing bytes instead. Hence we compute `maskBytes` of the 32 bytes vector, then we remove every odd bit reducing the mask value, so:

```
and = [0, 0, 0, 0, 0, 0, 0, 0, 1, 1, ... // the rest of 22 zeros ]
maskBytes = 768 // little-endian binary is 0000000011 (big-endian binary is 1100000000)
```

becomes:

```
and = [0, 0, 0, 0, 1, ... // the rest of 11 zeros ]
mask = 16 // little-endian binary is 00001 (big-endian binary is 10000)
```

We remove every odd bit using bitwise operations:

```csharp
private static int RemoveOddBits(int n)
{
  n = ((n & 0x44444444) >> 1) | (n & 0x11111111);
  n = ((n & 0x30303030) >> 2) | (n & 0x03030303);
  n = ((n & 0x0F000F00) >> 4) | (n & 0x000F000F);
  n = ((n & 0x00FF0000) >> 8) | (n & 0x000000FF);
  return n;
}
```

Next, we have to figure out the position of the first bit set to `1`, so we do more bitwise magic:

```csharp
private static int GetFirstBit(int n)
{
  return (int) Bmi1.TrailingZeroCount((uint) n);
}
```

Under the hood `TrailingZeroCount` performs the [TZCNT](https://www.felixcloutier.com/x86/tzcnt) instruction that counts the number of trailing least significant zero bits. In other words, it calculates the position of the first significant bit in the little-endian representation of `n` — exactly what we need.

Finally, if the candidate is not the string we are looking for, we clear the bit reducing the size of the mask:

```csharp
private static int ClearFirstBit(int n)
{
  return (int) Bmi1.ResetLowestSetBit((uint) n);
}
```

This intrinsic gets translated into the [BLSR](https://www.felixcloutier.com/x86/blsr) instruction that resets the lowest set bit.

In the C++ implementation `memcmp` is used for checking the candidate string against the needle. We don't have this goodie in C#, so we have to write a poor man's `memcmp` ourselves:

```csharp
private static unsafe bool Compare(char* source, int sourceOffset, char* dest, int destOffset, int length)
{
  for (var i = 0; i < length; i++)
  {
    if (source[sourceOffset + i] != dest[destOffset + i])
    {
      return false;
    }
  }

  return true;
}
```

Essentially we just check every candidate's character against the needle one by one. One detail to note here — we don't pass the first and the last indices of the candidate to this method as we already know that they match. Instead, we are passing the second and the second to last indices.

# Benchmarking

Time to compare all the implementations we gathered throughout this post. In the benchmark we will search for a needle located at the end of the 10k word haystack:

```
|     Method |      Mean |     Error |    StdDev |
|----------- |----------:|----------:|----------:|
|      Naive | 50.105 us | 0.3209 us | 0.3002 us |
|    IndexOf |  2.560 us | 0.0224 us | 0.0198 us |
|      RegEx | 33.651 us | 0.2659 us | 0.2357 us |
| Intrinsics |  9.595 us | 0.0948 us | 0.0841 us |
```

That's right, built-in `IndexOf` (that takes `Ordinal` as the `StringComparison` parameter) is the fastest because it exploits the `SpanHelpers.IndexOf` method which [uses hardware intrinsics internally](https://github.com/dotnet/runtime/blob/6b5850fa7cf476b66be9d01a3ed05f4484b01d3a/src/libraries/System.Private.CoreLib/src/System/SpanHelpers.Char.cs#L17) (and a better string search algorithm for sure). The benchmark was run under the following spec:

- macOS Catalina 10.15.4
- Intel Core i7-8569U CPU 2.80GHz (Coffee Lake)
- .NET Core SDK=3.1.200

# Possible Improvements

Nevertheless, if we want to improve the performance of our `IntrinsicsIndexOf` further we can take a look at the following approaches: instruction pipelining and memory alignment.

In the [Intel Intrinsics Guide](https://software.intel.com/sites/landingpage/IntrinsicsGuide/), each intrinsic has Latency and Throughput characteristics that depend on the CPU architecture. The former means how long it takes for the CPU to complete the instruction, the latter has to do with how many operations we can perform at once, each of them being in the different execution phase. For instance, `_mm256_loadu_si256` that we were using for loading data into a vector, has the latency of 1 and the throughput of 0.25 on the Haswell architecture. Meaning, we can run up to 4 instructions per 1 CPU cycle. We don't necessarily get the x4 performance because we are unlikely to have optimal conditions. This optimization applies to small loops only, so it should be suitable for our case. Using this knowledge and knowing our CPU architecture, we can unroll some vector instructions in the main block loop expecting at least some performance benefit.

According to Intel's [Developer Guide and Reference](https://software.intel.com/content/www/us/en/develop/documentation/cpp-compiler-developer-guide-and-reference/top/compiler-reference/intrinsics/data-alignment-memory-allocation-intrinsics-and-inline-assembly/alignment-support.html#alignment-support) aligning data should also improve the performance of intrinsics. In our code, we are not aware of memory alignment. Dealing with it, though, is not an easy job, but still doable. For example, we could search for the first aligned element and start with it processing previous elements without vectors. Applying this optimization we could use `LoadAlignedVector256` function. It gets translated to the `_mm256_load_si256` intrinsic which verifies that data is aligned.

# Conclusion

Indeed, the example we were going through is not very practical, but I hope it gives an overview of what hardware intrinsics are capable of. Since they are providing CPU specific functionality, we have to be aware of the hardware we run our code on. I guess you can see why the main rule of performance — always measure it — is especially important for intrinsics.

You can check out the code from this post on GitHub: [HaystacksNeedlesAndHardwareIntrinsics](https://github.com/timiskhakov/HaystacksNeedlesAndHardwareIntrinsics).

# Further Reading

- [Hardware Intrinsics in .NET Core](https://devblogs.microsoft.com/dotnet/hardware-intrinsics-in-net-core/)
- [Hardware intrinsic in .NET Core 3.0 - Introduction](https://fiigii.com/2019/03/03/Hardware-intrinsic-in-NET-Core-3-0-Introduction/)
- [Exploring .NET Core platform intrinsics](https://mijailovic.net/2018/06/06/sha256-armv8/)
- [Squeezing the Hardware to Make Performance Juice — Sasha Goldshtein](https://www.youtube.com/watch?v=5jKLVGI3B4g)
