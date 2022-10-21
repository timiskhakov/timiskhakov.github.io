---
layout: post
title: Numbers in Vectors
excerpt: Analyzing and implementing a SIMD algorithm for parsing a series of integers
---

Every now and then my friend [Alexei](https://github.com/AlexeiMatrosov) and I play a fun little game unofficially called "Can you make it faster?" We send each other a piece of code, usually something we've been working on recently, and try to figure out if it's possible to speed it up. Normally, this code solves a particular problem, works on a determined input, and has an existing solution that most likely runs in production. I [wrote](https://timiskhakov.github.io/posts/dnd-and-algorithms) about one such game a while ago. Since it's just for fun, it is allowed to use unsafe code, call external libraries written in other languages, cache things extensively, and exploit hardware-specific features — anything goes.

The other day, Alexei sent me a problem that looked innocent on the surface along with his solution and asked if I could make it faster. That led me quite deep into the rabbit hole of SIMD programming, but eventually I managed to find an approach that might be worth taking a look at. In this post, we are going to analyze this algorithm, implement it in C#, and see if our solution beats Alexei's. Before we dive into the weeds, though, let's properly formulate the problem.

## Problem

Let's say we have a string containing non-negative integers separated by commas, that we need to parse and return as an array of unsigned integers. For example, after processing the input:

```
"123,456,789"
```

the following array is returned:

```
[123, 456, 789]
```

The string is encoded in UTF-8. The range of possible values spans between `0` and `uint.MaxValue` which is 4,294,967,295. For simplicity, we assume that the input is valid, in other words:
- no other characters apart from commas and digits are included in the string
- no comma is followed by another comma
- the string starts and ends with a digit

Since we're going to implement our solutions in .NET, it's important to note that the type we return is `uint[]`, not `IEnumerable<uint>` or some other interface — that's part of the requirement.

## Naive Solution

First, let's start with a naive solution that we are going to use as a baseline:

```csharp
public uint[] Parse(string value)
{
  if (string.IsNullOrEmpty(value)) return Array.Empty<uint>();

  var result = new List<uint>();
  int start = 0, end;
  while ((end = value.IndexOf(',', start)) != -1)
  {
    result.Add(uint.Parse(value.AsSpan(start, end - start)));
    start = end + 1;
  }

  result.Add(uint.Parse(value.AsSpan(start)));

  return result.ToArray();
}
```

This implementation is probably the first that comes to mind once we get familiarized with the problem. First, we create the `result` for storing parsed numbers. Then we scan the string looking for a comma. Once we find it, we parse all the characters between the starting point and the comma, add the parsed value to the resulting `List`, and set the comma position to be a new starting point — rinse and repeat. Since our requirement dictates returning an array, not `List`, we call `ToArray` on the `result` at the very end.

This is, of course, a simple and easy-to-read solution, however, it can be improved time- and memory-wise.

## Optimized Solution

In the naive solution, there is a problem with memory allocations when adding elements to the `result`. `List` uses an array internally to store elements and creates a bigger array copying all values from the current one once its capacity is reached. Also, at the very end, we need to conform to the required return type by calling `ToArray`, effectively creating another array and copying all values from the `result`.

Alexei made an interesting optimization to avoid additional allocations and copying values between arrays:

```csharp
public uint[] Parse(string value)
{
  if (string.IsNullOrEmpty(value)) return Array.Empty<uint>();

  var commas = 0;
  foreach (var v in value)
  {
    if (v == ',') commas++;
  }

  var result = new uint[commas + 1];
  var start = 0;
  for (var i = 0; i < result.Length - 1; i++)
  {
    var end = start;
    while (value[end] != ',') end++;
    result[i] = uint.Parse(value.AsSpan(start, end - start));
    start = end + 1;
  }

  result[^1] = uint.Parse(value.AsSpan(start));

  return result;
}
```

First, we need to count the number of commas we have in the input string, so we would know how many numbers we need to parse. This way, we allocate the output array only once as we know its final size. Indeed, we need to do this additional pass through the array, but as we will soon see, it works faster and saves us some memory. Next, we apply a similar logic for scanning numbers as we did in the naive solution. Once we reach the last comma, we just parse the rest of the string.

Now, if we benchmark both solutions on different inputs we can see the benefits of this approach:

```
|    Method |     Value |             Mean |  Allocated |
|---------- |---------- |-----------------:|-----------:|
|     Naive | 123456789 |         43.10 ns |      104 B |
| Optimized | 123456789 |         24.94 ns |       32 B |
|           |           |                  |            |
|     Naive |      0-99 |      1,723.85 ns |     1608 B |
| Optimized |      0-99 |      1,216.44 ns |      424 B |
|           |           |                  |            |
|     Naive |   0-9,999 |    171,847.04 ns |   171424 B |
| Optimized |   0-9,999 |    148,908.98 ns |    40024 B |
|           |           |                  |            |
|     Naive | 0-999,999 | 22,523,932.60 ns | 12389476 B |
| Optimized | 0-999,999 | 21,352,869.17 ns |  4000081 B |
```

This solution is tightly optimized, so it would be hard to beat it. However, it only parses one number at a time, but what if we could parse several numbers at once? You guessed it right, I'm talking about using SIMD — single instruction, multiple data — processing.

## SIMD Solution: Analysis

We have [talked](https://timiskhakov.github.io/posts/haystacks-needles-and-hardware-intrinsics) about SIMD in the past, so I won’t get into details this time. Just to recap, modern x86 CPUs[^1] have various extensions that support processing multiple data with only one instruction. They provide additional registers for packed data and a set of instructions to operate on them. Most of the instructions are available for programmers through intrinsic functions that lately became available in .NET.

For our SIMD solution we are going to need a CPU that supports [AVX2](https://en.wikipedia.org/wiki/Advanced_Vector_Extensions), basically, any x86 processor that came out after 2013. Luckily for us, we don't need to invent an algorithm that enables vectorized parsing from scratch. Wojciech Muła wrote an excellent article [Parsing series of integers with SIMD](http://0x80.pl/articles/simd-parsing-int-sequences.html) that describes the algorithm and its implementation in C++ in detail. What we are going to do is analyze it and port the code to C#.

The core idea behind the SIMD approach is to read the input substring by substring, loading each of them into a vector, to shuffle the vector so that the numbers go one after another, omitting the commas, and to parse the numbers altogether.

We start by determining the size of the vector. In theory, the bigger it gets, the more data we can process at a time, but it's not that simple in practice. Since we know that the input string can only contain digits and commas, we will use bytes for storing characters, that is, one byte for one character. AVX2 allows us to use 256-bit (or 32-byte) registers, but there is a catch: a 256-bit register is partitioned into two 128-bit lanes. It's not a problem for most instructions, but when it comes to reordering elements within a vector representing the register's data, it's not possible to perform such an instruction cross-lane[^2]. So, it leaves us with a 128-bit (or 16-byte) vector that would hold our data.

Once we load a substring into the vector, we need to analyze what kind of data we have. For example, we might have something like this:

```
1,23,456,7890,12
```

At this point we don't care about particular digits, we may as well convert the character sequence into a pattern using `1` and `0` to denote digits and commas, respectively:

```
1011011101111011
```

Each number occurrence, like `1`, `11`, or `111`, forms a span that is of interest to us. Looking at the pattern we can deduct a few things:
- the amount of numbers (or spans)
- the size of the numbers, that is, how many digits each has
- how many bytes we can process

The last point is interesting. The pattern ends with two digits, so we can’t be sure that there aren't any following digits that would form a number. Thus, we can say that we only processed 14 bytes. If a pattern would end with a comma, like this:

```
1111011101101110
```

We could safely assume that we processed all 16 bytes. When we load the next substring into the vector, we would need to advance the input to the number of bytes we processed.

Next thing we need is to normalize the pattern, reorder and exclude some spans, and adjust the number of processed bytes if needed. For that, we are going to determine the maximum size of the spans and, if it's greater than 1, round it to the next power of two. This way we will get the maximum size of 1, 2, 4, 8, or 16. Using it, we can divide our vector into slots in one of the following combinations: 16 slots for 1-digit spans, 8 slots for 2-digit spans, 4 slots for 4-digit spans, 2 slots for 8-digit spans, or just 1-slot for a 16-digit span[^3].

Now we can reorder spans into slots. Our example pattern would be reordered into the following vector:

```
0001001101111111
```

Or, converting the pattern back to the sequence:

```
0001002304567890
```

If the number size was less than the maximum size, we'd just padded it with zeros, so `1` becomes `0001`, `23` — `0023`, and `456` — `0456`.

Knowing that we deal with four 4-digit numbers, we can choose the right method for parsing the numbers.

There are two things left to cover: how exactly we shuffle an arbitrary pattern into a vector of equally-sized numbers, and the implementation of the parsing methods for each slot size. We will explore both in the next section.

## SIMD Solution: Implementation

Now that we have familiarized ourselves with the theory, let's finally open an editor and implement the algorithm. It's going to be a long ride, so let's break it down into a few subsections:
- Overview
- Counting Commas
- Parsing a Vector
- Precalculating Data
- Parsing Methods

### Overview

We start with the main `Parse` method:

```csharp
public unsafe uint[] Parse(string value)
{
  if (string.IsNullOrEmpty(value)) return Array.Empty<uint>();

  var result = new uint[CountCommas(value) + 1]; // (1)
  var processed = 0;
  var amount = 0;
  fixed (char* c = value)
  {
    Span<uint> output = stackalloc uint[8]; // (2)
    while (processed <= value.Length - 16)
    {
      var vector = LoadInput(c + processed); // (3)
      var (p, a) = ParseVector(vector, output);
      for (var i = 0; i < a; i++)
      {
        result[amount + i] = output[i]; // (4)
      }
        
      processed += p;
      amount += a;
    }
  }

  for (var i = amount; i < result.Length - 1; i++) // (5)
  {
    var end = processed;
    while (value[end] != ',') end++;
    result[i] = uint.Parse(value.AsSpan(processed, end - processed));
    processed = end + 1;
  }

  result[^1] = uint.Parse(value.AsSpan(processed));

  return result;
}
```

There's a lot to take in, so let's tackle main points one by one:
1. As in the optimized solution, we need to count numbers first, so that we know how many numbers we need to parse and allocate an array of proper size. Since the amount of numbers is just the amount of commas plus 1, we'll count commas as it's easier to do (we leave the implementation for the next subsection). We also need to initialize two counters, `processed` and `amount`. The former tracks the number of bytes processed to advance the input and load a correct substring into a vector. The latter serves for counting parsed numbers that were added to the `result` array.
2. Next, we allocate a small array to re-use it as an output from the method that parses a vector. Since the maximum amount of numbers we can theoretically parse is only eight, there won't be any harm if we allocate it on the stack.
3. Then we load a substring into a vector and parse it. `ParseVector` returns the number of bytes processed within the vector and the amount of numbers it placed into the `output` array.
4. The numbers are then added to the `result` array and we advance both counters, `processed` and `amount`.
5. Once we finish the SIMD processing, we process the rest of the string by applying the same approach we used in the optimized solution.

Now let's have a look at the methods we used throughout `Parse`. We'll explore `CountCommas` first.

### Counting Commas

Similarly to the optimized solution, in the SIMD approach, we also do two passes through the string: the first is to count an amount of numbers — or an amount of commas as they're connected — and the second is to parse them. While we are employing SIMD for the actual processing, why not speed up the counting of commas as well?

```csharp
private static readonly Vector128<byte> Commas = Vector128.Create((byte)',');

private static unsafe uint CountCommas(string value)
{
  uint result = 0;
  var i = 0;
  fixed (char* c = value)
  {
    for (; i < value.Length - Vector256<ushort>.Count; i += Vector256<ushort>.Count)
    {
      var vector = LoadInput(c + i);
      var match = Sse2.CompareEqual(Commas, vector);
      var mask = Sse2.MoveMask(match);
      result += Popcnt.PopCount((uint)mask);
    }
  }

  for (; i < value.Length; i++)
  {
    if (value[i] == ',') result++;
  }

  return result;
}
```

First of all, we need to initialize the `result`. Next, we go through the input string dividing it into substrings and loading each into a 128-bit vector using the `LoadInput` method. We compare the input vector against a vector of commas to get a mask. All we need to do now is to count the number of bits in the mask — that would be the number of commas in the substring. Once we are done with vector processing, we count the rest of the commas in a simple loop.

For loading a substring into a vector, we need to perform a small trick implemented in `LoadInput`:

```csharp
private static readonly Vector128<byte> RawMask = Vector128.Create(
  0, 2, 4, 6, 8, 10, 12, 14,
  0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80, 0x80);

private static unsafe Vector128<byte> LoadInput(char* c)
{
  var raw = Avx.LoadDquVector256((ushort*)c);
  var bytes = raw.AsByte();
  var lower = Ssse3.Shuffle(bytes.GetLower(), RawMask);
  var upper = Ssse3.Shuffle(bytes.GetUpper(), RawMask);
  
  return Vector128.Create(lower.GetLower(), upper.GetLower());
}
```

In .NET, `char` is a 2-byte value that can be represented as `ushort`. The problem is, we need a `byte` representation of the substring. So we load the substring into a 256-bit vector `raw` and represent it as `bytes`. For each character, this representation provides an additional byte we don't need. For example, character `1` becomes `{0, 1}`, or `{48, 49}` in ASCII, due to the 2-byte nature of `char`. So, we shuffle the lower and upper lanes separately using the `RawMask` to put the significant byte first and get rid of `48`. Then we combine the lower parts of the lanes into a new 128-bit vector.

A quick illustration for the method would be the following:

```
c         = 1,23,456,7890,12
raw       = <'1', ',', '2', '3', ',', '4', '5', '6', 
             ',', '7', '8', '9', '0', ',', '1', '2'> 
bytes     = <48, 49, 48, 44, 48, 50, 48, 51,
             48, 44, 48, 52, 48, 53, 48, 54,
             48, 44, 48, 55, 48, 56, 48, 57,
             48, 48, 48, 44, 48, 49, 48, 50>
lower     = <49, 44, 50, 51, 44, 52, 53, 54,
             48, 48, 48, 48, 48, 48, 48, 48>
upper     = <44, 55, 56, 57, 48, 44, 49, 50,
             48, 48, 48, 48, 48, 48, 48, 48>
result    = <49, 44, 50, 51, 44, 52, 53, 54,
             44, 55, 56, 57, 48, 44, 49, 50>
```

The same `LoadInput` is used in `Parse` described in the previous subsection.

### Parsing a Vector

Okay, we've reached the big boss of the solution, a method that parses the numbers. Let's explore it:

```csharp
private static readonly Vector128<sbyte> ZerosAsSByte = Vector128.Create((byte)'0').AsSByte();
private static readonly Vector128<sbyte> ColonsAsSByte = Vector128.Create((byte)':').AsSByte();

private (int, int) ParseVector(Vector128<byte> input, Span<uint> output)
{
  var t0 = Sse2.CompareLessThan(input.AsSByte(), ZerosAsSByte);
  var t1 = Sse2.CompareLessThan(input.AsSByte(), ColonsAsSByte);
  var pattern = Sse2.AndNot(t0, t1);
  var mask = Sse2.MoveMask(pattern);
  var block = _blocks[mask];
  var shuffled = Ssse3.Shuffle(input, pattern.Mask);

  switch (pattern.NumberSize)
  {
    case 1:
      Parse1DigitNumbers(shuffled, pattern.Amount, output);
      break;
    case 2:
      Parse2DigitNumbers(shuffled, pattern.Amount, output);
      break;
    case 4:
      Parse4DigitNumbers(shuffled, pattern.Amount, output);
      break;
    case 8:
      Parse8DigitNumbers(shuffled, pattern.Amount, output);
      break;
    case 16:
      ParseSingeNumber(shuffled, output);
      break;
    default:
      throw new InvalidOperationException();
  }

  return (pattern.Processed, pattern.Amount);
}
```

The method takes in a vector we prepared in the previous subsection and a `Span<uint>` that would hold parsed numbers. As denoted earlier, it returns a number of processed bytes and an amount of parsed numbers. The gist of the method is to find a pattern of spans (we discussed it in the **Analysis** section) and shuffle it so that we would know the size and amount of the numbers as well as the number of processed bytes. Then based on the size we would employ one of the parsing methods.

First, we need to create two temporary vectors by performing a signed compare of the input vector against a vector of zeros and the input vector against a vector of colons. Followed by performing the `Sse2.AndNot` (or `_mm_andnot_si128`) instruction which NOTs the first vector and ANDs it with the second, we get the span pattern containing digits only.

Next, we compute a pattern's mask and obtain a precalculated block that holds some interesting data we need for the parsing:
- the size of the numbers rounded to the next power of two
- the amount of numbers
- bytes processed
- a shuffle mask that would tell us how to re-arrange the elements in the vector so that it becomes parsable

We will come to the precalculated blocks in an upcoming subsection.

In order to actually do the in-vector reordering, we need to use the `Ssse3.Shuffle` (or `_mm_shuffle_epi8`) instruction which takes the vector and a mask and produces a new vector shuffled according to the mask.

Finally, in a big giant `switch` we select a parsing function and pass our shuffled vector to it.

### Precalculating Data

Effectively, each pattern we could get is uniquely represented by a 16-bit number. Since we know all possible patterns — `2^16` which is 65,536 — we can precalculate all the information we need in advance: the amount of numbers, their maximum size, bytes the pattern processed, and the shuffle mask. All that form a structure that we call a `Block`:

```csharp
public class Block
{
  private const int VectorSize = 16;

  public int NumberSize { get; }
  public int Amount { get; }
  public int Processed { get; }
  public Vector128<byte> Mask { get; }
}
```

I'll omit code for calculating block's data for brevity, but you can check it out using the repo link at the end of the post. Since the blocks are going to be precalculated on the application startup, we don't need to squeeze performance out of their calculations, so we'll try to keep them as simple as possible.

Let's just have a look at how a block is formed by using our example from the **Analysis** section. We had the following sequence:

```
1,23,456,7890,12
```

That was converted into the pattern:

```
1011011101111011
```

The pattern has four spans (`1`, `11`, `111`, and `1111`), the number size of 4, and 14 bytes processed. Now we need to check if the 16-byte vector fits all the spans: `Amount * NumberSize  <= VectorSize`. In our example, we are good as `4 * 4 <= 16`, but if we had to deal with a pattern like this:

```
1101110111111101
```

We would have three spans (`11`, `111`, and `1111111`), the number size of 8, and 15 bytes processed. Which clearly won't fit the vector: `3 * 8 = 24`. Then we would have to remove the last span, do the number size calculation again, and adjust the bytes processed. In that case, we would get two spans (`11` and `111`), the number size of 4 (as we need to round it to the next power of two), and 7 bytes processed. The spans fit the vector: `2 * 4 = 8`, but the rest of the pattern has to go to the next batch of processing.

Coming back to our example, we need to pad the spans with zeros:

```
0001001101111111
```

And then form a new vector substituting `0`s with `0x80` and `1`s with the initial positions of the numbers, which turns out to be the mask:

```
<0x80, 0x80, 0x80, 0, 0x80, 0x80, 2, 3, 0x80, 5, 6, 7, 9, 10, 11, 12>
```

Once we know how to calculate the block's data, we need to cache all possible blocks in the parser's class:

```csharp
private readonly Dictionary<int, Block> _blocks = new();

public SimdParser()
{
  for (ushort i = 0; i < ushort.MaxValue; i++)
  {
    _blocks.Add(i, new Block(i));
  }
}
```

Since we go through all possible `ushort` values, we will also create blocks for patterns that would not be valid from the problem's perspective, such as this one that has multiple consecutive commas:

```
1000000001111111
```

Of course, we could exclude those, but since we precalculate the blocks, it won't matter much if we have a bit more data in the cache.

### Parsing Methods

Last but not least, we have five parsing methods that we need to go over. Let's start with the simplest one, `Parse1DigitNumbers`:

```csharp
private static readonly Vector128<byte> Zeros = Vector128.Create((byte)'0');

private static void Parse1DigitNumbers(Vector128<byte> vector, int amount, Span<uint> output)
{
  var t0 = Sse2.SubtractSaturate(vector, Zeros);
  for (var i = 0; i < amount; i++)
  {
    output[i] = t0.GetElement(i);
  }
}
```

Parsing 1-digit numbers is easy: we have to subtract zero's ASCII code from the vector and, knowing the amount of numbers, get them one by one.

Parsing 2-, 4-, and 8-digit numbers has a few additional steps involving the `MultiplyAddAdjacent` intrinsic. Let's have a look at `Parse2DigitNumbers` as an example:

```csharp
private static readonly Vector128<sbyte> Mul10 = Vector128.Create(10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1);

private static void Parse2DigitNumbers(Vector128<byte> vector, int amount, Span<uint> output)
{
  var t0 = Sse2.SubtractSaturate(vector, Zeros);
  var t1 = Ssse3.MultiplyAddAdjacent(t0, Mul10);
  for (var i = 0; i < amount; i++)
  {
    output[i] = (uint)t1.GetElement(i);
  }
}
```

First things first, we subtract the zeros to get a temporary vector `t0`. Then we perform `MultiplyAddAdjacent` (or `_mm_maddubs_epi16`) that would multiply the temporary vector with `Mul10` element-wise and then add an element to its next neighbor producing a new vector `t1`. Just to illustrate that by using example values for `t0`, we'll get the following `t1`:

```
t0    = <1, 2, 3, 4, 5, 6, 7, 8, 9, 0, 1, 1, 1, 1, 1, 1>
Mul10 = <10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1, 10, 1>
t1    = <12, 34, 56, 78, 90, 11, 11, 11>
```

All we need to do at the end is to obtain a correct amount of numbers from `t1` one by one.

`Parse4DigitNumbers` and `Parse8DigitNumbers` look somewhat similar, so we'll skip their implementations, but you can check them out using a link to the repo at the end of the post.

Finally, to parse a number that has more than 8 digits, we resort to non-SIMD processing:

```csharp
private static void ParseSingeNumber(Vector128<byte> vector, Span<uint> output)
{
  Span<char> chars = stackalloc char[16];
  var start = 0;
  for (var i = 0; i < chars.Length; i++)
  {
    var element = vector.GetElement(i);
    chars[i] = (char)element;
    if (element == 0) start++;
  }

  output[0] = uint.Parse(chars[start..]);
}
```

First, we allocate a small array on the stack. Then we count the digits as we don't know how many of them the number has. Finally, we use plain `uint.Parse` providing a span that represents the number.

## Benchmarks

We've gone a long way, but now it's time to see if it was worth it. Since we're using hardware-specific code it's important to note that the benchmarks were run on the following specs:

- BenchmarkDotNet=v0.13.2, OS=Windows 11 (10.0.22621.674)
- Intel Core i7-10750H CPU 2.60GHz, 1 CPU, 12 logical and 6 physical cores
- .NET SDK=6.0.402, .NET Runtime 6.0.10

```
|    Method |     Value |             Mean |  Allocated |
|---------- |---------- |-----------------:|-----------:|
|     Naive | 123456789 |         43.91 ns |      104 B |
| Optimized | 123456789 |         23.48 ns |       32 B |
|      Simd | 123456789 |         30.52 ns |       32 B |
|           |           |                  |            |
|     Naive |      0-99 |      1,625.96 ns |     1608 B |
| Optimized |      0-99 |      1,177.62 ns |      424 B |
|      Simd |      0-99 |        616.87 ns |      424 B |
|           |           |                  |            |
|     Naive |   0-9,999 |    169,395.66 ns |   171424 B |
| Optimized |   0-9,999 |    147,788.80 ns |    40024 B |
|      Simd |   0-9,999 |     95,413.96 ns |    40024 B |
|           |           |                  |            |
|     Naive | 0-999,999 | 22,554,080.83 ns | 12389475 B |
| Optimized | 0-999,999 | 20,605,416.04 ns |  4000080 B |
|      Simd | 0-999,999 | 15,935,536.46 ns |  4000080 B |
```

Looking at the benchmarks we can draw an obvious conclusion: the more digits the numbers have, the less effective the SIMD solution becomes. This comes as no surprise as we did optimize it for parsing multiple numbers at the same time: the less digits the numbers have, the more of them we can pack in a vector.

Let's face it, in most cases we don't want to have this kind of code in our applications: it's unsafe, not portable across different CPU architectures, and not easy to maintain. Unless we need to have it due to some performance requirements. In this case, it's good to know that we have a way to squeeze more performance out of code trading it for something else, like portability and maintainability.

There's an old saying that goes with every performance-related blog post: don't trust them, always measure performance on your hardware yourself.

You can check out the code from this post on GitHub: [ParsingNumbers](https://github.com/timiskhakov/ParsingNumbers).

## Footnotes

[^1]: Processors that use other instruction set architectures, such as ARM, also support SIMD operations, but in the context of this post, we're only interested in the x64 ISA.

[^2]: It's possible to achieve cross-lane shuffling using several instructions, but to keep things simple and follow the blog idea we won't go this direction.

[^3]: It's actually impossible to get more than 8 1-digit numbers as there has to be a comma separating each number.