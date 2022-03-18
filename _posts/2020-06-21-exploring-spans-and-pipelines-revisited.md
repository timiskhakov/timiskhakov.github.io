---
layout: post
title: Exploring Spans and Pipelines Revisited
excerpt: Revisiting a previous post on spans and pipelines and improving parsing performance
---

A while ago we [explored](exploring-spans-and-pipelines) the usage of `Span<T>` and `Pipelines` to speed up file parsing. However, as David Fowler has [rightly pointed out](https://github.com/timiskhakov/ExploringSpansAndPipelines/issues/1) we might have had a problem with stack allocations if we had long lines in the file. Considering the data I had to work with and trying to keep the post as simple as possible, I've come up with a workaround by putting a line length limit into the method that processes the line. Now the time has come, let's remove the workaround and replace it with a proper solution instead.

## Introducing ArrayPool

I don't want to copy the whole code from the previous post, so let's take a quick look at the method that contains the limit:

```csharp
private static Videogame ProcessSequence(ReadOnlySequence<byte> sequence)
{
  if (sequence.IsSingleSegment)
  {
    return Parse(sequence.FirstSpan);
  }

  var length = (int) sequence.Length;
  if (length > LengthLimit)
  {
    throw new ArgumentException($"Line has a length exceeding the limit: {length}");
  }

  Span<byte> span = stackalloc byte[(int)sequence.Length];
  sequence.CopyTo(span);

  return Parse(span);
}
```

We might notice that the problem with the limit comes from the fact that `ReadOnlySequence<byte>` consists of segments. If it has one segment only, we parse it directly, otherwise, we allocate a byte array on the stack for further processing. To deal with long lines we need a way to either use a heap allocated array to copy the sequence data to it or process the sequence segment by segment.

Let's take the first path and allocate an array to get the data out of the sequence. However, instead of just creating it we will use [`ArrayPool<T>`](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1) that provides a resource pool for reusing arrays of `T`. The usage of the pool is quite simple: once we rent and use an array, we need to return it back to the pool. The pool provides `Rent` and `Return` methods for these actions. Now it's time to refactor `ProcessSequence` and use `ArrayPool<T>`:

```csharp
private static readonly ArrayPool<byte> ArrayPool = ArrayPool<byte>.Shared;

private static Videogame ProcessSequence(in ReadOnlySequence<byte> sequence)
{
  var length = (int) sequence.Length;
  var array = ArrayPool.Rent(length);
  try
  {
    sequence.CopyTo(array);
    return LineParserImproved.Parse(array.AsSpan().Slice(0, length));
  }
  finally
  {
    ArrayPool.Return(array);
  }
}
```

We rent an array providing `sequence.Length` as its minimum length, then we copy sequence content to it, parse the data and return the array back to the pool.

As you probably noticed we don't convert bytes to chars here. Instead, we create a `Span<byte>` on top of the array and pass it down to the line parser, which in theory should save us some processing time. (To verify it we will do benchmarking at the end of the post.) Clearly, we need to refactor its `Parse` method as well.

## Introducing Utf8Parser

One detail to note before we go further: since we are going to change the signature of the line parser, we don't need to follow the contract of `ILineParser` we were sticking to previously. Frankly, I ran out of good names for the parsers (not that I had them before, though), so I named the new implementation simply `LineParserImproved`:

```csharp
public static class LineParserImproved
{
  public static Videogame Parse(ReadOnlySpan<byte> bytes)
  {
    return new Videogame
    {
      Id = ParseGuid(ref bytes),
      Name = ParseString(ref bytes),
      Genre = (Genres) ParseInt(ref bytes),
      ReleaseDate = ParseDateTime(ref bytes),
      Rating = ParseInt(ref bytes),
      HasMultiplayer = ParseBool(ref bytes)
    };
  }

  // ...
}
```

Each `Parse*Type*` method heavily relies on the [`Utf8Parser`](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.text.utf8parser) that has a method for parsing almost every built-in primitive type:

```csharp
private static Guid ParseGuid(ref ReadOnlySpan<byte> bytes)
{
  if (!Utf8Parser.TryParse(bytes, out Guid value, out var consumed)) throw new ArgumentException(nameof(bytes));
  if (bytes.Length >= consumed + 1) bytes = bytes.Slice(consumed + 1);
  return value;
}

private static int ParseInt(ref ReadOnlySpan<byte> bytes)
{
  if (!Utf8Parser.TryParse(bytes, out int value, out var consumed)) throw new ArgumentException(nameof(bytes));
  if (bytes.Length >= consumed + 1) bytes = bytes.Slice(consumed + 1);
  return value;
}

private static DateTime ParseDateTime(ref ReadOnlySpan<byte> bytes)
{
  if (!TryParseExactDateTime(bytes, out var dateTime, out var consumed)) throw new ArgumentException(nameof(bytes));
  if (bytes.Length >= consumed + 1) bytes = bytes.Slice(consumed + 1);
  return dateTime;
}

private static bool ParseBool(ref ReadOnlySpan<byte> bytes)
{
  if (!Utf8Parser.TryParse(bytes, out bool value, out var consumed)) throw new ArgumentException(nameof(bytes));
  if (bytes.Length >= consumed + 1) bytes = bytes.Slice(consumed + 1);
  return value;
}
```

Of course, for string parsing we need to come up with a bit different, but quite similar approach:

```csharp
private static string ParseString(ref ReadOnlySpan<byte> bytes)
{
  var position = bytes.IndexOf((byte) '|');
  var value = Encoding.UTF8.GetString(bytes.Slice(0, position));
  if (bytes.Length >= position + 1) bytes = bytes.Slice(position + 1);
  return value;
}
```

As you might have noticed we don't use `Utf8Parser` in `ParseDateTime`. Its `TryParse` method for `DateTime` parsing [supports several formats](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.text.utf8parser.tryparse#System_Buffers_Text_Utf8Parser_TryParse_System_ReadOnlySpan_System_Byte__System_DateTime__System_Int32__System_Char_), but, sadly, a `DateTime` property in our data doesn't belong to those, it has a date part only, `yyyy-MM-dd`. Although, knowing the format, nothing stops us from implementing our own parser [based](https://github.com/dotnet/runtime/blob/4f9ae42d861fcb4be2fcd5d3d55d5f227d30e723/src/libraries/System.Private.CoreLib/src/System/Buffers/Text/Utf8Parser/Utf8Parser.Date.cs) on the built-in one:

```csharp
private static bool TryParseExactDateTime(in ReadOnlySpan<byte> bytes, out DateTime value, out int consumed)
{
  value = default;
  consumed = 0;

  if (bytes.Length < 10) return false;

  var digit1 = bytes[0] - 48u; // 48u == '0'
  var digit2 = bytes[1] - 48u;
  var digit3 = bytes[2] - 48u;
  var digit4 = bytes[3] - 48u;
  if (digit1 > 9 || digit2 > 9 || digit3 > 9 || digit4 > 9) return false;

  var year = 1000 * digit1 + 100 * digit2 + 10 * digit3 + digit4;

  if (bytes[4] != (byte)'-') return false;

  var digit5 = bytes[5] - 48u;
  var digit6 = bytes[6] - 48u;
  if (digit5 > 9 || digit6 > 9) return false;

  var month = 10 * digit5 + digit6;

  if (bytes[7] != (byte)'-') return false;

  var digit8 = bytes[8] - 48u;
  var digit9 = bytes[9] - 48u;

  var day = 10 * digit8 + digit9;
  if (digit8 > 9 || digit9 > 9) return false;

  value = new DateTime((int) year, (int) month, (int) day, 0, 0, 0, DateTimeKind.Utc);
  consumed = 10;

  return true;
}
```

## Benchmarking

We got quite good results in the previous post. Let's benchmark the new implementation and see if it gets better or worse. We use the same benchmark in which we parse a 500k line file into the application's data models:

```
|                  Method |       Mean |    Error |   StdDev | Allocated |
|------------------------ |-----------:|---------:|---------:|----------:|
|              FileParser | 1,428.9 ms | 28.27 ms | 57.75 ms | 375.22 MB |
|         FileParserSpans | 1,169.2 ms | 30.55 ms | 60.07 ms | 230.55 MB |
| FileParserSpansAndPipes |   646.6 ms | 20.75 ms | 47.08 ms |  78.75 MB |
|      FileParserImproved |   454.7 ms | 13.66 ms | 40.27 ms |  78.75 MB |
```

The benchmark was run under the following spec:

- macOS 10.15.5
- Intel Core i7-8569U CPU 2.80GHz (Coffee Lake)
- .NET Core SDK=3.1.200

Not only we got rid of the line length limit we also increased the speed of parsing! You can find implementations of `LineParserImproved` and `FileParserImproved` in the same repository: [ExploringSpansAndPipelines](https://github.com/timiskhakov/ExploringSpansAndPipelines).
