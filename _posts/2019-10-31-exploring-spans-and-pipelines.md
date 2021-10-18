---
layout: post
title: "Exploring Spans and Pipelines"
---

The other day I was working on a client project when I stumbled upon a ticket that required me to move some functionality from an old legacy system to our .NET Core backend. The functionality itself was fetching a text file from the network, parsing it into application data structures, and saving them to the database. Sounds quite easy, doesn't it? So, I wrote some code that solved the problem and moved on...

...until later I started to dig into `Span<T>` and `Pipelines` and thought: heck, that text file parsing task is a good real life problem to try all these shiny new toys on!

## Problem

Let's make up a similar problem with a completely arbitrary domain. Say, we have an application that works with... I dunno, video games (I'm a bit into this topic, so, the domain is not really arbitrary). Imagine we have a model that describes a video game as:

```csharp
public class Videogame
{
  public Guid Id { get; set; }
  public string Name { get; set; }
  public Genres Genre { get; set; }
  public DateTime ReleaseDate { get; set; }
  public int Rating { get; set; }
  public bool HasMultiplayer { get; set; }
}
```

`Genres` is just an enum:

```csharp
public enum Genres
{
  Action = 1,
  Adventure = 2,
  Rpg = 3,
  Shooter = 4,
  Sports = 5,
  Strategy = 6
}
```

We also have a giant list of games somewhere that we have to fetch and parse into a bunch of classes described above. To make things simpler, imagine that "somewhere" is just a PSV file on a disk (that is, a pipe-separated file, just like CSV, but with a pipe `|` instead of a comma). A video game is represented by a single line in the file:

```
38e27dea-1d7d-4279-be97-e29d53a8af89|F.E.A.R.|4|2005-10-18|90|False
```

Meaning: it's a shooter called F.E.A.R. released on 18.10.2005 that has no multiplayer, but does have an ID and rating.

Let’s say there are 500 000 lines in this file in total, the maximum length of each line is 150 characters. Our goal is to parse the file into a list of data structures located in the memory as fast as possible using as little memory as we can.

(I created a file containing fake data with a simple data generator. If you are going to check out the source code, don't be surprised by the test data.)

## First Approach

The problem is simple, right? First, let's try to write a naïve line parser:

```csharp
public interface ILineParser
{
  Videogame Parse(string line);
}

public class LineParser : ILineParser
{
  public Videogame Parse(string line)
  {
    var parts = line.Split('|');

    return new Videogame
    {
      Id = Guid.Parse(parts[0]),
      Name = parts[1],
      Genre = Enum.Parse<Genres>(parts[2]),
      ReleaseDate = DateTime.ParseExact(
        parts[3],
        "yyyy-MM-dd",
        DateTimeFormatInfo.InvariantInfo,
        DateTimeStyles.None),
      Rating = int.Parse(parts[4]),
      HasMultiplayer = bool.Parse(parts[5])
    };
  }
}
```

We will need the interface a bit later for the file parser. Also, important to note: since it's demo code, let's assume we only deal with ideal data and not bother with handling corner cases, catching possible exceptions, and validating models. (Although in real life we would do that, right? ...right?)

Second, we need something to read the file and feed each line to the line parser, so, let's write a simple file parser:

```csharp
public interface IFileParser
{
  Task<List<Videogame>> Parse(string file);
}

public class FileParser : IFileParser
{
  private readonly ILineParser _lineParser;

  public FileParser(ILineParser lineParser)
  {
    _lineParser = lineParser;
  }

  public async Task<List<Videogame>> Parse(string file)
  {
    var videogames = new List<Videogame>();

    using (var stream = File.OpenRead(file))
    using (var reader = new StreamReader(stream))
    {
      while (!reader.EndOfStream)
      {
        var line = await reader.ReadLineAsync();
        var videogame = _lineParser.Parse(line);
       	videogames.Add(videogame);
      }
    }

    return videogames;
  }
}
```

Nice and clean! The code indeed solves the problem.

## Introducing Spans<T>

Can we make the code above better? Well, it depends on what "better" means. Though, there is at least one thing that could possibly be improved. We do a lot of memory allocation as we work with strings. In order to reduce them let's introduce [`Span<T>`](https://docs.microsoft.com/en-us/dotnet/api/system.span-1), a new type that's allocated on the stack, and use it to write a new implementation of `ILineParser`:

```csharp
public class LineParserSpans : ILineParser
{
  public Videogame Parse(string line)
  {
    var span = line.AsSpan();

    return Parse(span);
  }

  public static Videogame Parse(ReadOnlySpan<char> span)
  {
    // Don't worry, we will increment this value in ParseChunk
    var scanned = -1;
    var position = 0;

    var id = ParseChunk(ref span, ref scanned, ref position);
    var name = ParseChunk(ref span, ref scanned, ref position);
    var genre = ParseChunk(ref span, ref scanned, ref position);
    var releaseDate = ParseChunk(ref span, ref scanned, ref position);
    var rating = ParseChunk(ref span, ref scanned, ref position);
    var hasMultiplayer = ParseChunk(ref span, ref scanned, ref position);

    return new Videogame
    {
      Id = Guid.Parse(id),
      Name = name.ToString(),
      Genre = (Genres)int.Parse(genre),
      ReleaseDate = DateTime.ParseExact(releaseDate, "yyyy-MM-dd", DateTimeFormatInfo.InvariantInfo),
      Rating = int.Parse(rating),
      HasMultiplayer = bool.Parse(hasMultiplayer)
    };
  }

  private static ReadOnlySpan<char> ParseChunk(ref ReadOnlySpan<char> span, ref int scanned, ref int position)
  {
    scanned += position + 1;

    position = span.Slice(scanned, span.Length - scanned).IndexOf('|');
    if (position < 0)
    {
      position = span.Slice(scanned, span.Length - scanned).Length;
    }

    return span.Slice(scanned, position);
  }
}
```

All right, things got a bit tricky here. We still operate on a string representation of a video game, but instead of splitting it into multiple strings we create a `Span<char>` out of it. After that we move from one pipe to another until the end creating a slice (that is, a part of the span) each time we move. Since we know the string format, we can rely on slice positions. Yes, it might look a bit clunky, but we [don't have](https://github.com/dotnet/corefx/issues/26528) something like `Split()` method on `Span<T>` as of now, so we have to do it manually.

Another thing worth mentioning: we make `Parse(ReadOnlySpan<char> span)` public and static because we will need it a bit later.

At this point we have two different line parser implementations — it's time to do some benchmarking. Let's install [`BenchmarkDotNet`](https://github.com/dotnet/BenchmarkDotNet), write some code to run different implementations, and... relax on a chair waiting for it to do its job. While waiting let me describe my setup. This and other benchmarks will be run under the following spec:

- macOS Mojave 10.14.6
- Intel Core i7-7567U CPU 3.50GHz (Kaby Lake)
- .NET Core SDK=3.0.100

Bear in mind, though, that your results might be different. Oh, speak of the devil, here is the result:

```
|          Method |     Mean |    Error |   StdDev |  Gen 0 | Gen 1 | Gen 2 | Allocated |
|---------------- |---------:|---------:|---------:|-------:|------:|------:|----------:|
|      LineParser | 756.0 ns | 14.61 ns | 17.39 ns | 0.1945 |     - |     - |     408 B |
| LineParserSpans | 581.2 ns | 12.61 ns | 17.68 ns | 0.0496 |     - |     - |     104 B |
```

Well, the result is indeed positive, but to be honest, it is not that exciting, is it? But what about the big picture? Let's run some benchmarks on file parsers and feed our 500k line file to them:

```
|          Method |       Mean |    Error |   StdDev |       Gen 0 |      Gen 1 |     Gen 2 | Allocated |
|---------------- |-----------:|---------:|---------:|------------:|-----------:|----------:|----------:|
|      FileParser | 1,156.2 ms |  8.31 ms |  7.77 ms | 114000.0000 | 40000.0000 | 5000.0000 | 375.13 MB |
| FileParserSpans |   862.6 ms | 16.51 ms | 16.22 ms |  50000.0000 | 15000.0000 | 4000.0000 | 230.51 MB |
```

Well, that's something, if we are talking about memory consumption.

## Introducing Pipelines

There is still a problem with our code. Can you guess what it is? If you said that `LineParserSpans` still works with strings, you are absolutely right. Wouldn't it be faster to use `Parse(ReadOnlySpan<char> span)` from `LineParserSpans` and skip the string part? Indeed it would. Although, that requires us to break the existing contract and somehow extract a line from the file as `ReadOnlySpan<char>`.

In order to do that, let me bring yet another toy to the existing code — `System.IO.Pipelines`. It's a new library in the .NET world designed for high performance IO. Kestrel, ASP.NET Core's web server, uses `System.IO.Pipelines` under the hood. Pipelines are similar to streams, but the Pipelines library is faster as it uses `Span<T>` and its API is clearer.

But let's go back to our code. We will make a new implementation of `IFileParser` and here is what we will do:

1. This time 'round we will read the file chunk by chunk, instead of string by string.
2. We will store each chunk in a buffer represented by `ReadOnlySequence<byte>`.
3. Then we will read lines from this buffer, also represented by `ReadOnlySequence<byte>`, and parse them into our `Videogame` model.
4. We might have a situation when we have a line located between buffers: one part — at the end of one buffer, and another — at the beginning of the next buffer. Unlike streams, `Pipelines` does buffer management itself, which allows us to resolve such cases easily.
5. To keep things simple we will use a default buffer size.

Let's see how we can do all this:

```csharp
public class FileParserSpansAndPipes : IFileParser
{
  private const int LengthLimit = 256;

  public async Task<List<Videogame>> Parse(string file)
  {
    var result = new List<Videogame>();
    using (var stream = File.OpenRead(file))
    {
      PipeReader reader = PipeReader.Create(stream);
      while (true)
      {
        ReadResult read = await reader.ReadAsync();
        ReadOnlySequence<byte> buffer = read.Buffer;
        while (TryReadLine(ref buffer, out ReadOnlySequence<byte> sequence))
        {
          var videogame = ProcessSequence(sequence);
          result.Add(videogame);
        }

        reader.AdvanceTo(buffer.Start, buffer.End);
        if (read.IsCompleted)
        {
          break;
        }
      }
    }
    return result;
  }

  private static bool TryReadLine(ref ReadOnlySequence<byte> buffer, out ReadOnlySequence<byte> line)
  {
    var position = buffer.PositionOf((byte)'\n');
    if (position == null)
    {
      line = default;
      return false;
    }

    line = buffer.Slice(0, position.Value);
    buffer = buffer.Slice(buffer.GetPosition(1, position.Value));

    return true;
  }

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

  private static Videogame Parse(ReadOnlySpan<byte> bytes)
  {
    Span<char> chars = stackalloc char[bytes.Length];
    Encoding.UTF8.GetChars(bytes, chars);

    return LineParserSpans.Parse(chars);
  }
}
```

So, we create a pipe connected to our good ol' friend `FileStream`. Pipelines allows us to read the file chunk by chunk using a buffer. `TryReadLine` reads the buffer and finds lines. Then we pass each line to `ProcessSequence(ReadOnlySequence<byte> sequence)`. With that method, we check whether the sequence contains a whole line, or two parts of a line that came from different chunks. In the first case, we just parse the line. In the second case, we merge the parts into a new line, and then parse it. Bear in mind, before parsing we should convert `Span<byte>` into `Span<char>`. We do that in `Parse(ReadOnlySpan<byte> bytes)`. (Remember that method in `LineParserSpans` we made public and static? We use it here.).

While reading and parsing a chunk we have to tell Pipelines how much data we consumed and examined. We do it by calling `reader.AdvanceTo(buffer.Start, buffer.End)`. That way we allow Pipeline to do its buffer management.

Ok, it's time for final benchmarking:

```
|                  Method |       Mean |    Error |   StdDev |       Gen 0 |      Gen 1 |      Gen 2 | Allocated |
|------------------------ |-----------:|---------:|---------:|------------:|-----------:|-----------:|----------:|
|              FileParser | 1,378.5 ms | 15.78 ms | 14.76 ms | 133000.0000 | 60000.0000 | 25000.0000 | 375.34 MB |
|         FileParserSpans |   905.5 ms | 17.71 ms | 25.96 ms |  52000.0000 | 18000.0000 |  8000.0000 | 230.56 MB |
| FileParserSpansAndPipes |   640.5 ms |  7.69 ms |  6.82 ms |  15000.0000 |  7000.0000 |  3000.0000 |  78.77 MB |
```

Pretty good, huh?

## Conclusion

As they always say in performance related posts: please, don't rush to rewrite all your code in production using new cool libraries. Performance is a feature, not something that's given. Sometimes the current solution is good enough. But if speed or memory efficiency are something you need in your applications, you might want to have a look at `Span<T>` and `Pipelines`.

You can check out this code and do your own benchmarks based on it: [ExploringSpansAndPipelines](https://github.com/timiskhakov/ExploringSpansAndPipelines/).

## Further Reading

- [Exploring Spans and Pipelines Revisited](exploring-spans-and-pipelines-revisited)
- [Memory and Span usage guidelines](https://docs.microsoft.com/en-us/dotnet/standard/memory-and-spans/memory-t-usage-guidelines/)
- [Span](https://adamsitnik.com/Span/)
- [System.IO.Pipelines: High performance IO in .NET](https://devblogs.microsoft.com/dotnet/system-io-pipelines-high-performance-io-in-net/)
- [System.IO.Pipelines — a little-known tool for lovers of high performance](https://habr.com/ru/post/466137/)
