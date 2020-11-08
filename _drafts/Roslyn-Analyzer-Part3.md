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

The input to this method is the [`CodeFixContext`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codefixes.codefixcontext?view=roslyn-dotnet). It's a `Struct` which contains a collection of objects we can use to perform some important actions.

We start of by first getting the `SyntaxRoot` for the document where we've have at least one diagnostic from those we specified in the `FixableDiagnosticIds`.

We then want to get some additional information around the code for which we've reported the diagnostic. We do this by accessing the `context.Diagnostics` collection's first element. We use the first element because in our case our `FixableDiagnosticIds` registers to address only one diagnostic and we tend to always expect this array to have one element!

{: .box-note} 
**Note:** The note on the description of `context.Diagnostics` states  that all the diagnostics in the collection apply to the same [Span](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codefixes.codefixcontext.span?view=roslyn-dotnet#Microsoft_CodeAnalysis_CodeFixes_CodeFixContext_Span). This means the same area of code to fix. This translated that potentially we might have multiple diagnostics reported for the same area of code that we can provided fixes for through a single Fix Provider.
{: .box-note}

Let's look at these specific lines:

``` csharp
var diagnosticSpan = diagnostic.Location.SourceSpan;

// Find the type declaration identified by the diagnostic.
var declaration = root.FindToken(diagnosticSpan.Start).Parent.AncestorsAndSelf().OfType<TypeDeclarationSyntax>().First();
```

From the reported diagnostic we get the [`Location.SourceSpan`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.location.sourcespan?view=roslyn-dotnet#Microsoft_CodeAnalysis_Location_SourceSpan) property which we use to look through the Syntax Root and in our case find the first matching `TypeDeclarationSyntax`. This gives us the `SyntaxNode` which contains the lower-case type declaration.

> The Sample problem we are looking for treats any Type Declaration that contains at least one lower-case character as a problem that is reported by a diagnostic and we will see, fixed by upper-casing the entire name of the Type.

After we find the node we can finally call [`RegisterCodeFix`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codefixes.codefixcontext.registercodefix?view=roslyn-dotnet). If for some reason we can't apply code fixes to all of our diagnostics based on certain conditions we can opt out to not register the Code Fix after checking our conditions.

We use [`CodeAction.Create`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.codeactions.codeaction.create?view=roslyn-dotnet) with the following important line:

``` csharp
createChangedSolution: c => MakeUppercaseAsync(context.Document, declaration, c)
```

We set the `createChangedSolution` parameter to be a function that can be called with some of the parameters we supply. It's an `async` method with `c` as the cancellation token and the `context.Document` and `declaration` of the type.

The alternative to `createChangedSolution` is the `createChangedDocument` which would allow us to change only the single document with the reported diagnostic. But if we thing about the nature of this sample problem we will want to change the usages of the Type Declaration in other documents to match/use the "fixed" uppercase name of the Type.

{: .box-note}
**Note:** For another overview of registering fix providers have a look at this [link](https://docs.microsoft.com/en-us/archive/msdn-magazine/2015/february/csharp-adding-a-code-fix-to-your-roslyn-analyzer#fixablediagnosticids-property) which goes through a different sample problem based off of the same template.
{: .box-note}

Next let's take a quick look at how we actually change code/documents in the solution!

### Applying Code Fixes

We've now registered the `MakeUppercaseAsync(context.Document, declaration, c)` method that can change the solution. This method **is** going to modify source code! And is invoked from two contexts:

1. **Previewing:** When the user wants to action the Diagnostic - through the Quick Actions or through hovering on the Squigly lines in the IDE. This does not apply the fix but calls the method to generate the new code so that the user can **preview** the fix
2. **Applying:** When the user, happy with the preview actually applies the fix. This is the point where we start modifying source code!

The `MakeUppercaseAsync` method is pretty straightforward and uses some new Roslyn API features as we will see. Of note is also the return type of `Task<Solution>`. The method is asynchronous as it can be invoked through the Visual Studio IDE. It returns a solution, as we will be making changes to all references to the Type in all of the documents in the solution:

``` csharp
private async Task<Solution> MakeUppercaseAsync(Document document, TypeDeclarationSyntax typeDecl, CancellationToken cancellationToken)
{
   // Compute new uppercase name.
   var identifierToken = typeDecl.Identifier;
   var newName = identifierToken.Text.ToUpperInvariant();

   // Get the symbol representing the type to be renamed.
   var semanticModel = await document.GetSemanticModelAsync(cancellationToken);
   var typeSymbol = semanticModel.GetDeclaredSymbol(typeDecl, cancellationToken);

   // Produce a new solution that has all references to that type renamed, including the declaration.
   var originalSolution = document.Project.Solution;
   var optionSet = originalSolution.Workspace.Options;

   var newSolution = await Renamer.RenameSymbolAsync(document.Project.Solution, typeSymbol, newName, optionSet, cancellationToken).ConfigureAwait(false);

   // Return the new solution with the now-uppercase type name.
   return newSolution;
}
```

What we initially do is get the identifier name (the type name) as defined at the moment with some lower-case character through the `TypeDeclarationSyntax`. Using that it's simple to create a "fixed" name by transforming the identifier.Text property with `ToUppwerInvariant':

``` csharp
// Compute new uppercase name.
var identifierToken = typeDecl.Identifier;
var newName = identifierToken.Text.ToUpperInvariant();
```

The following four lines are just fetching and setting up the objects we will need to fully generate our new solution. Here we use the `SemanticModel` to fetch the `TypeSymbol`. The symbol through the semantic model gives us information about this Type Declaration that can be used beyond just this document. One usage is, as we see, for the `Renamer.RenameSymbolAsync`.

The documentation for the [`Renamer`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.rename.renamer?view=roslyn-dotnet) class and the [`RenameSymbolAsync`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.rename.renamer.renamesymbolasync?view=roslyn-dotnet#Microsoft_CodeAnalysis_Rename_Renamer_RenameSymbolAsync_Microsoft_CodeAnalysis_Solution_Microsoft_CodeAnalysis_ISymbol_System_String_Microsoft_CodeAnalysis_Options_OptionSet_System_Threading_CancellationToken_) is a bit sparse but based on the API we can see that it generates a new solution.

In our case the `typeSymbol` will be assigned the `newName` in the active `document.Project.Solution`. The option set parameter is the default one from the original solution.

We see that the `Renamer` class is very powerful and very suited for this specific example scenario. We will now look at the fix for the Multiple Method invocations which is a bit more involved and relies more on us manually generating code which we then use to replace the multiple calls!

## Fixing Multiple Method Invocations
