---
layout: post
tags: [.net, roslyn, refactoring, analysis, analyzer, syntax tree, semantic, symbols, diagnostic, arguments]
title: Adventures with Roslyn .NET, Part 3 - Fix Provider
author: emir_osmanoski
comments: true
image: /images/2020-11-02-Roslyn-Syntax-Analyzer-Part-2/00_logo.png
cover-img: /images/2020-11-02-Roslyn-Syntax-Analyzer-Part-2/000_analysis_cover.png
share-img: /images/2020-11-02-Roslyn-Syntax-Analyzer-Part-2/00_logo.png
meta-description: An introduction to a basic Analyzer for the Multiple Method call Invocation
---

This is going to be the final part of this Roslyn Adventure and will cover a
simple fix provider for the sample problem we identified and anylzed in the
previous parts:

For full context, the following links (will) point to the previous articles:

- [Introduction](https://blog.emirosmanoski.mk/2020-06-11-Roslyn-Syntax-Analyzer-Multiple-Method-Calls/)
- [Analyzer](https://blog.emirosmanoski.mk/2020-11-02-Roslyn-Roslyn-Analyzer-Part2/)
- Fix Provider

This part is going to be short overview of one potential way to implement a fix
provider that is going to modify the calls in our code and replace the multiple
calls using the same parameters with a single call, store it's value and re-use
the stored value.

As in the previous article that looked at the analyzer we will start to get aquatinted
with the Fix Provider Roslyn API through the sample provided with with the `Analyzer with Code Fix Provider` Template

## Template Default setup

### Intro and Solution

The previous article already fully covered the Problem & Analyzer for the
template project. We will only recap that the DiagnosticId property of the
issue/diagnostics reported by the analyzer we looked at is :

``` csharp
public const string DiagnosticId = "UpperCaseType";
```

### Fix Provider Overview

Similar to how we can register `Analyzer` we start with a Fix Provider Type that
inherits from the base
[`CodeFixProvider`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codefixes.codefixprovider?view=roslyn-dotnet):


``` csharp
[ExportCodeFixProvider(LanguageNames.CSharp, Name = nameof(UpperCaseTypeCodeFixProvider)), Shared]
public class UpperCaseTypeCodeFixProvider : CodeFixProvider
{
   public sealed override ImmutableArray<string> FixableDiagnosticIds
   {
      get { return ImmutableArray.Create(UpperCaseTypeAnalyzer.DiagnosticId); }
   }

   public sealed override FixAllProvider GetFixAllProvider()
   {
      return WellKnownFixAllProviders.BatchFixer;
   }

   // ... MORE CODE TO FOLLOW
}
```

{: .box-note} 
**Note:**  Depending on the Templates and Visual Studio versions
used the Template will either contain the Fix Provider in the common projet with
the Analyzer or it will be in it's own project in the solution:
`UpperCaseType.CodeFixes`
{: .box-note}

There are several key points from these initial lines of code:

1. We inherit from `CodeFixProvider` 
2. We `ExportCodeFixProvider` by decorating the class with the attribute
3. We define for which Diagnostics we can provide fixes by overriding the `FixableDiagnosticIds` property
4. We define the FixAllProvider - which determines how this fix is applied to multiple errors. This uses the default Batch Fixer. More information [here](https://github.com/dotnet/roslyn/blob/master/docs/analyzers/FixAllProvider.md)


### Registering Code Fixes

The next part of the code overview will look at the `RegisterCodeFixesAsync` method. It's an Async which can be used to register code fixes for the Diagnostics our Fix Provider has registered for via `FixableDiagnosticIds`.


``` csharp
public sealed override async Task RegisterCodeFixesAsync(CodeFixContext context)
{
   var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken).ConfigureAwait(false);

   // TODO: Replace the following code with your own analysis, generating a CodeAction for each fix to suggest
   var diagnostic = context.Diagnostics.First();
   var diagnosticSpan = diagnostic.Location.SourceSpan;

   // Find the type declaration identified by the diagnostic.
   var declaration = root.FindToken(diagnosticSpan.Start).Parent.AncestorsAndSelf().OfType<TypeDeclarationSyntax>().First();

   // Register a code action that will invoke the fix.
   context.RegisterCodeFix(
         CodeAction.Create(
            title: CodeFixResources.CodeFixTitle,
            createChangedSolution: c => MakeUppercaseAsync(context.Document, declaration, c),
            equivalenceKey: nameof(CodeFixResources.CodeFixTitle)),
         diagnostic);
}
```

{: .box-note} 
**Note:** This is a method that is called each time an engineer "starts" to bring up the contextual menu with all the potential fixes for a given diagnostic/"issue". This method allows us to register one or multiple potential code fixes that the user can decide to apply.
{: .box-note}

The input to this method is the [`CodeFixContext`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codefixes.codefixcontext?view=roslyn-dotnet). It's a collection of b




## Sample Problem - ACTUAL

We saw in the previous part covering the Analyzer that we can report a `Diagnostic` object that can appear in the VS IDE to the users of the extension.

The key part is that the Diagnostics we report have a Diagnostic ID Associated
with them which makes them easy to identify and potentially address. In the case
of our problem statement - we set the diagnostic id to:

``` csharp
public const string DiagnosticId = "MultipleMethodCallDiagnosticAnalyzer";
```