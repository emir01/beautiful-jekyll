---
layout: post
tags: [C#, programming, .net, puzzles, challenge, aoc, advent, of, code]
title: Advent of Code - First 10 Days - Impressions
author: emir_osmanoski
comments: true
published: true
image: /images/2022-12-11-Ten-Days-Of-AoC/000_Logo.png
cover-img: /images/2022-12-11-Ten-Days-Of-AoC/00_Dotnet6Logo.png
share-img: /images/2022-12-11-Ten-Days-Of-AoC/000_Logo.png
meta-description: Advent of Code - First 10 Days - My Experience
---

Advent of Code (AoC) is a once a year coding challenge that consists of 25 puzzles
over 25 days starting from December.

It got my attention for the first time, after the last edition ended, through some
online discussions. I looked at the challenges and they seemed interesting, but
at that point I did not really have the time to explore more.

So, I made a little promise to myself: I was going to try and follow the next
years edition as long as possible and as long as I had the time.

Lucky me I do have the time, and after the initial 10 days I want step aside and
share some thoughts on why AoC could be a very interesting experience for any
engineer.

# Contents

- [Contents](#contents)
- [The Puzzles](#the-puzzles)
- [So, why Participate?](#so-why-participate)
  - [Fun? :partying\_face:](#fun-partying_face)
  - [Exploring Language Features :fire:](#exploring-language-features-fire)
  - [Exploring Communities :house\_with\_garden:](#exploring-communities-house_with_garden)
  - [Flow Mode :brain:](#flow-mode-brain)
  - [Creativity and Problem Solving :bulb:](#creativity-and-problem-solving-bulb)
- [Final Words :checkered\_flag:](#final-words-checkered_flag)

# The Puzzles

The AoC puzzles are holiday themed, released daily and increasing in difficulty
as the month goes on. So far, except the first 2 puzzles, they have all had 2
parts, each of which when solved offers a little yellow star as a reward.

Any programming language can be used and the solution is a string that
the puzzle solving code needs to calculate from a fairly large input.

The puzzles are very unique and interesting, telling a story, this year so far
mostly about the challenges Santa's elves are facing in preparation for
Christmas.

For some more details and a better explanation on how everything is setup
(including the leader-board aspects) you can head on over to the official about
page:

- [Advent of Code 2022 - About](https://adventofcode.com/2022/about)

One thing I can say straight away, is that I'm amazed at the ingenuity of the
puzzles and how well they fit the theme and create a story.

For example, the very last [10th puzzle](https://adventofcode.com/2022/day/10)
deals with "fixing" a communication device given to you by Santa's Elves, in one
of the early days - which is now broken after falling in the water on day nine.

You do this by dealing with it's CPU, looking at instructions and figuring out how
those interact with a simple CRT monitor. 

The puzzle is split into two parts with the final solution requiring you to draw
what the CRT monitor would output after all the input has been processed.

What makes this even better is that the author includes links to Wikipedia
articles referencing the problems you are trying to figure out. 

For example the CRT Wikipedia Page does contain a very interesting animation
illustrating what the code you can write could attempt to do to get to the solution:

{:refdef: style="text-align: center;"}
![CRT](/images/2022-12-11-Ten-Days-Of-AoC/00_CRT.gif)
{:refdef}

# So, why Participate?

Let's look at some aspects which might make AoC something worth giving a shot! 

## Fun? :partying_face:

Sure, it might not be everyone's cup of tea, but for me these puzzles have been
like little escape rooms where you have to figure your way out. This is
even more enhanced by the way the puzzles are described, as mentioned following a connected story.

It is like you are on a little adventure, and right now I'm far enough to want to
see how the story ends.

I think they also offer a little hit of dopamine every time you figure something
out and every time you have an *aha!* moment as you are testing your solutions.

It all culminates in the end when the answer is submitted and you get your
stars. On top of everything these is a nice bit of gamification, seeing all the
star's you've collected on your dashboard. :star:

{:refdef: style="text-align: center;"}
![Dashboard](/images/2022-12-11-Ten-Days-Of-AoC/01_Dashboard.png)
{:refdef}

{: .box-note}
As you can see from the screenshot I still haven't started with the 11th day.
That comes after this post is finished (hopefully) :grin:
{: .box-note}

## Exploring Language Features :fire:

I decided to try the puzzles using **C#**. It's a language that has been evolving a
lot recently with a lot of new features and syntax sugar added all the time:

- [C# 11 - What's New!](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-11)

When working in teams, I find that sometimes it might be difficult to introduce
and/or explore some of these features. Especially if the team has an already
established way of doing certain things. 

Having a consistent code base and conventions, arguably is more important than
always rushing to try these new features, more so if we take into account the
costs/benefits and context of having teams make bigger changes.

{: .box-note} 
Additionally, even though we might read about the new features as they come out,
if we don't get to use them we run the risk of starting new project using the
same old comfortable toolbox. 
{: .box-note}

On top of that, a lot of the work we do is usually abstracted by frameworks and
specific high level business cases, so we don't get to spend time exploring some
of the language basic/low-level features which can only benefit us and our
understanding of these tools we use.

Which is why a challenge like AoC is a perfect use-case to try the latest and
greatest version and features of your favorite language and to get them
ingrained in your mind. 

This way, when starting  new projects we can have a much better perspective of
what can be useful for us, and we can better setup everything and have our teams
on board to move on to the newer versions.

Here are just some examples of what I got to explore and sometimes try while
going through the first 10 puzzles:

1. [Record Types](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/tutorials/records)
   1. Used to quickly define typed objects to use while parsing inputs and problem solving.
2. [File Scoped Namespace Declaration](https://learn.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-10#file-scoped-namespace-declaration)
   1. Just one of the recent improvements that make C# files more readable and improve DevEx.
3. [Tuple Types](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/value-tuples)
   1. Used for representing problem space and data.
   2. Even though not a new concept I rarely get to use it in day to day work so jumped on the opportunity to explore the concept a bit more.
4. Json Serialization with [System.Text.Json](https://learn.microsoft.com/en-us/dotnet/standard/serialization/system-text-json/overview)
   1. Used in Debugging purposes
   2. System.Text.Json has recently seen a lot of performance improvements and in some use cases is recommended as a substitute to [Json.NET](https://www.newtonsoft.com/json).
5. [Math.NET Numerics](https://numerics.mathdotnet.com/)
   1. Knowing that I would probably need to deal with a lot of matrix representations of problem spaces I explored Math/Max libraries.
   2. So far matrix operations have been simple enough to be able to use native C# Multi Dim Arrays - More on this next
6. [Yield Return](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/statements/yield)
   1. Used to process [CPU state](https://adventofcode.com/2022/day/10) after each cycle while at the same time allowing CPU State to fully complete.
   
---

One thing that was bit of an *"aha!"* moment, which I enjoyed quite a bit, was having
to deal with arrays and matrices. 

It's definitely something I don't get to do daily (probably never in the past 5
years if not more so), so I'll admit :satisfied:, I could not recall from the
top of my head how the multi-dim array syntax was defined in C#. Problem made
even worse by years of using `List<T>`, and separately Javascript and
Typescript. 

So here it is now in all its glory, so hopefully I never forget again:

``` csharp
int[,] trees = new int [rows, columns];
```

{: .box-note}
 As an addition I discovered (re-discovered?) that the C#
 multidimensional arrays also have a very different API than the one I was most
used to in, for example, a language like JavaScript. 

I could not just do: `trees.Length` to get the number of rows. Instead you get the length of a
specific dimension via: `trees.GetLength(0)` - indicating the first dimension. 
{: .box-note}

Finally, there were some use-cases where I needed to generate multiple objects
to [track a rope with multiple knots](https://adventofcode.com/2022/day/9)
moving around. 

I wanted to make the solution work for any knot size, so a way to generate a
list of my Knot/BrideLoc objects was necessary. A very interesting way to do
that is using the Enumerable class and some of the helper methods:

``` csharp
var knots = Enumerable.Range(0, numberOfKnots).Select(x => new BridgeLoc(0, 0)).ToList();
```

Even though there is also a `.Repeat(object, x)` method, you quickly learn that
it just repeats the same reference `x` - which causes all sorts of strange bugs! :beetle:

In hindsight it's something that should be obvious unless it was 1AM and you had
a very long day! :sleeping:

## Exploring Communities :house_with_garden:

I knew when starting that I wanted to focus on solving the actual problems and
not, at least this year, spend time on setting up a dev/test harness for the
puzzles.

That was my first introduction to the awesome community around the event. I
found this great repository with a lot of resources and specifically starter templates:

- [Awesome Advent of Code](https://github.com/Bogdanp/awesome-advent-of-code)

I ended up using the following template:

- [C# Advent of Code Template](https://github.com/eduherminio/AdventOfCode.Template)

The template is based on another community repository:
[AoCHelper](https://github.com/eduherminio/AoCHelper) and together they both
provide output formatting and performance measurements:

{:refdef: style="text-align: center;"}
![Template](/images/2022-12-11-Ten-Days-Of-AoC/02_Template.gif)
{:refdef}

I also discovered a lot of, new to me, content creators through Twitter
discussions usually under the
[#AoC2022](https://twitter.com/hashtag/AOC2022?src=hashtag_click) hashtag!

It's always fun to see how others approach some of these problems and even the
lengths they take to further challenge themselves. 

For example, I referenced the rope and knots puzzle previously and here is
someone actually visualizing the output:

- [Day09 - Visual - 36 Rope knots/segments](https://twitter.com/maartengm/status/1601176187233259521)

Finally, if you are from Macedonia :macedonia:, we can't wrap up this
section without mentioning the following channel that streams the challenge.

Worth a check to see how others brainstorm and think about the problems and
solutions :slightly_smiling_face:

-  [https://www.youtube.com/@swekster](https://www.youtube.com/@swekster)

## Flow Mode :brain:

So far, the puzzles have been clear enough, fun enough and just challenging
enough to very often get me into [Flow Mode.](https://en.wikipedia.org/wiki/Flow_(psychology))

What I noticed is that the challenge hasn't been just the difficulty of each
puzzle.At the beginning part of it  was embarking on this journey and seeing how
far I could get before starting to really get stuck on some of the problems.

There is also the challenge of understanding the actual problems. I had a case
where on one of the easier puzzles I completely misunderstood what was the ask.

But I was still super focused on trying to figure out why my solution was not
accepted and working through the requirements.

It was all in **the zone**

I think the more often we get into that state of mind the easier it is for us go
back to it when also doing other, maybe not so interesting things, or when we
would not be naturally up to it. 

So, I appreciate the opportunity to be able to "exercise that muscle" as much as
possible through these puzzles.

To be honest, at some point, I'm know I will hit **big wall.** I've been reading
that as the month goes on some of the puzzles get very hard.

And in "Flow theory", that might be a problem. When a task is hard enough to
potentially become frustrating it will impact the ability to get into that state
of mind.

Funnily enough, that is something to look forward to. The puzzles can be broken
down into different parts: parsing inputs, creating problem spaces with data
structures, research, brute forcing, breaking down the problem and
optimizations.

Each of these offers an opportunity to focus, and hopefully if done correctly,
and maybe with a bit of luck they would all contribute to a solution.

At the very least getting fully stuck will always offer a chance to reach out to
the community and learn something new!

{:refdef: style="text-align: center;"}
![Flow](/images/2022-12-11-Ten-Days-Of-AoC/03_Flow.jpeg)
{:refdef}

## Creativity and Problem Solving :bulb:

As mentioned in the previous section, the puzzles can be broken down into
multiple parts. And each one of those can offer a creative outlet, we don't
usually get in our day to day responsibilities. 

We have teams we work with, conventions processes and frameworks to follow,
which is expected to some point. So AoC offers a very frictionless way to be
creative.

In what way you might ask? 

Well, usually when approaching the puzzles I initially take an "instinctive"
approach to parse the input, represent the problem space and then create the
solutions. 

Some of the questions you could ask yourself would be along the lines of: Would I need a Matrix,
just some Coordinates or a complex model to represent the world? What would I
need to keep track of to get to the solution. Do I then need to perform some
sort of search over my results to build the final output? 

Sometimes in this first phase it's also very useful to visualize the puzzle.
Example from the [Day 9](https://adventofcode.com/2022/day/9) puzzle:

{:refdef: style="text-align: center;"}
![Whiteboard](/images/2022-12-11-Ten-Days-Of-AoC/04_AoC9_Whiteboard.png)
{:refdef}

In this way this first pass would offer an opportunity to stretch those creative
muscles, resulting in classes called `CpuInstruction`, `Crt`, `TreeSurroundings`
and all the behavior that goes with them.

{: .box-note}
Most of the solutions probably don't require models to that
extent, but I enjoyed defining them and find they help with the problem solving
that happens using them. 
{: .box-note}

Once the first part of the puzzle was solved and the second was revealed I'd
find that the initial solution was not quite supportive enough to be able to use
the same models and problem space to solve both parts..

So this would provide a chance to back to the drawing board with some more
information and to **refactor** what was already there. Offering another chance
to be creative and at the very least a chance to practice refactoring skills.

Long story short, my commit history for some of these problems sure tells
stories of triumphs from the first attempt, to horrible mistakes and large
reworks and multiple attempts to solve the problem.

{: .box-note} 
The nature of the challenge does not require any refactoring and
redesigns and you can solve the problems in any way you want. You can spend days
wiring the perfect most maintainable code or you could hack something together
and still get to the solution.
And that I think is one of the other great things about AoC :satisfied: 
{: .box-note}

# Final Words :checkered_flag:

This has been my journey so far with [Advent of Code
2022](https://adventofcode.com/) and above are my thoughts on why I think it's
been an enjoyable experience.

I still have 15 days ahead of me, including Day 11 - which is already open and
waiting for me! Time flies! :joy: 

I'm really looking forward to seeing what the future puzzles will be, and **I hope
I can make it** to the end and see those elves through to Christmas. :christmas_tree: 

If maybe some of the points above convinced you to give it a try the link is
here:

- [Advent of Code 2022](https://adventofcode.com/)

Finally, I also decided to keep my solution repository public. The code - following the template mentioned above, can be found here:

- [GitHub Repository - Advent of Code 2022 - C#](https://github.com/emir01/AoC2022)