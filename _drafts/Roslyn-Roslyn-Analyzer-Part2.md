---
layout: post
tags: [.net, roslyn, refactoring, analysis, analyzer, syntax tree]
title: Adventures with Roslyn .NET, Part 2 - Analyzer
author: emir_osmanoski
comments: true
image: /images/2020-06-11-Roslyn-Syntax-Analyzer-Multiple-Method-Calls/00_logo.png
cover-img: /images/2020-06-11-Roslyn-Syntax-Analyzer-Multiple-Method-Calls/000_cover.png
share-img: /images/2020-06-11-Roslyn-Syntax-Analyzer-Multiple-Method-Calls/00_logo.png
meta-description: An introduction to a basic Analyzer
---

We are now going to look at the Analyzer.

## Analyzer Basic Setup

In the previous post we mentioned that Roslyn offers a set of API's which allow
us to hook into the .NET Compiler pipeline that runs in Visual Studio.

This pipeline which includes Syntax Analysis is constantly running as it's
needed for some of the IDE features to work. For example the Syntax Highlighting
feature needs to know which parts of the text are variables and which parts are
methods so it can color them differently.

> Note: The Syntax Analysis building Syntax Trees is always running in the background. It analyzes the source code in our files and for every part of the text makes a decision on the type of the text.

Building on top of this  we can use the Roslyn APIs to hook into certain points
in this pipeline. We will be basically saying the following:

- Roslyn, please tell me when you identify a certain type of Syntax while you are processing this source file.

That is going to be enough for us to know for the purposes of what we want to achieve.

## Template Default setup

Let's have a look at how some of the above concept are covered in project generated by default from the template.

> The template generates a project that addresses a fictional code smell of lowercase names for types.
> For example when defining a class `public class Animal` the analyzer and fix provider will propose we change the class name to `public class ANIMAL`

The first thing we do to register an analyzer is going to be inherit from the
`DiagnosticAnalyzer` class and decorate using the Analyzer class with the
`DiagnosticAnalyzer` attribute and setting the language to C#:

``` csharp
[DiagnosticAnalyzer(LanguageNames.CSharp)]
public class BlogAnalyzerAnalyzer : DiagnosticAnalyzer
```

This allows us to override the `Initialize` method where we can actually let Roslyn know what we are interested in:

``` csharp
public override void Initialize(AnalysisContext context)
{
    // TODO: Consider registering other actions 
    // that act on syntax instead of or in addition to symbols
    //
    // See https://github.com/dotnet/roslyn/blob/master/docs/analyzers/Analyzer%20Actions%20Semantics.md for more information
    context.RegisterSymbolAction(AnalyzeSymbol, SymbolKind.NamedType);
}
```

The initialize method which runs when our Analyzer boots up is going to use the `AnalysisContext` ro register an `Action` with Roslyn.

The example uses `RegisterSymbolAction`. The description of the method states the following:

```
Register an action to be executed at completion of semantic analysis of an ISymbol with an appropriate Kind. A symbol action reports Diagnostics about ISymbols.
```

At this point we can make a difference between at least two  types of analysis we can conduct on our code base. In other words we can register actions we want performed an

1. Syntax Analysis
   1. Deals with the structure of the code by looking at how code constructs are structured and organized to form our program.
   2. Primary deals with Syntax Nodes and Syntax Tokens.
   3. Even though we can analyze multiple files at one given time we can only look at a single file.
2. Semantic Analysis
   1. Deals with the meaning behind the syntax.
   2. Among other things it's primary construct are the Symbols based on the ISymbol interface.
   3. The semantic analysis can span multiple files, for example we can get information for variables with types defined in other files.


## Additional Resources:

https://joshvarty.com/learn-roslyn-now/