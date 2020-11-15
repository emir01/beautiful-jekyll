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

In our case the `typeSymbol` will be assigned the `newName` in the active
`document.Project.Solution`. The option set parameter is the default one from
the original solution.

We see that the `Renamer` class is very powerful and very suited for this
specific example scenario. We will now look at the fix for the Multiple Method
invocations which is a bit more involved and relies more on us manually
generating code which we then use to replace the multiple calls!

## Fixing Multiple Method Invocations

Let's now have a look at the Fix Provider for the problem covered in this series. It follows the same pattern and initial setup by registering what the `FixableDiagnosticIds` are:

``` csharp
public sealed override ImmutableArray<string> FixableDiagnosticIds =>ImmutableArray.Create(Analyzer.DiagnosticId);

```

Our `RegisterCodeFixesAsync` is also very similar to the template:

``` csharp
public sealed override async Task RegisterCodeFixesAsync(CodeFixContext context)
{
   var root = await context.Document.GetSyntaxRootAsync(context.CancellationToken).ConfigureAwait(false);

   // getting first because we are only listening to a single diagnostic here
   // !important - we might have multiple diagnostics of the same type here for different places and methods in the code 
   var diagnostic = context.Diagnostics.First();

   var diagnosticSpan = diagnostic.Location.SourceSpan;

   // Find the type invocation identified by the diagnostic.
   var invocation = root.FindToken(diagnosticSpan.Start).Parent.AncestorsAndSelf()
         .OfType<InvocationExpressionSyntax>().First();

   // Register a code action that will invoke the fix.
   context.RegisterCodeFix(
         CodeAction.Create(
            title,
            c => CombineMethodUsageAsync(context.Document, invocation, c),
            title),
         diagnostic);
}
```

The important differences to note here is that we are looking for a
`InvocationExpressionSyntax` on which we will register the Code Fix, instead of
the `TypeDeclarationSyntax` as in the template. 

Additionally we register a Code Action called `CombineMethodUsageAsync` on just
the current active document, instead of the `solution`.

Let's move on to the `CombineMethodUsageAsync` which is a somewhat longer method.

{: .box-warning}
**Note:** The `Code Action` fix we are about to explore is not optimized and will contain some Syntax traversal and search code that might seem familiar with the [Analyzer](https://blog.emirosmanoski.mk/2020-11-02-Roslyn-Roslyn-Analyzer-Part2/) from Part 2.
{: .box-warning}

{: .box-warning}
**Additionally**, similar to the Analyzer it is a fairly simplified solution and sometimes might provide fixes that result in errors. This is something that a Code Fix should never do and some of these scenarios will be pointed out as we look at examples.
{: .box-warning}

### Intro to CombineMethodUsageAsync

The general approach to providing the code fix is going to be:

1. Get the full list of all invocations of the function/method we want to address within the given code block.
2. Look at the first method/function invocation as it appears in the method block.
3. If the result of the first invocation is stored within a variable don't do anything.
4. If the result of the first invocation is not stored in a variable - introduce a variable that will store the result.
5. Go over the remaining invocations of the method/function and replace them withe the variable defined either in 3) **or** 4).

So, lets see what we need to setup in step 1) in order to do our initial check in step 2):

``` csharp
private async Task<Document> CombineMethodUsageAsync(Document contextDocument,
            InvocationExpressionSyntax invocationRequestingFix,
            CancellationToken cancellationToken)
{
   // need to get all occurrences of the invocation of our method
   var expressionName = invocationRequestingFix.Expression.ToString();
   var documentEditor = await DocumentEditor.CreateAsync(contextDocument, cancellationToken);

   // find all of the Invocation Expression matching the expression name
   var originalInvocationsMatchingFixRequestInvocation = documentEditor
      .OriginalRoot
      .DescendantNodes()
      .OfType<InvocationExpressionSyntax>()
      .Where(x => x.Expression.ToString().Equals(expressionName))
      .Where(
         // only find the expression that are within the same code block
         x=>
         x.FirstAncestorOrSelf<SyntaxNode>(t=>t.IsKind(SyntaxKind.Block))
         == invocationRequestingFix.FirstAncestorOrSelf<SyntaxNode>(t => t.IsKind(SyntaxKind.Block)))
      .ToList();

   // if there is only one invocation
   if (originalInvocationsMatchingFixRequestInvocation.Count <= 1) return contextDocument;

   // assume the first invocation is assigned to a variable and will be skipped
   // variable is determined in next line of code.
   var originalInvocationsToSkipWhenFixing = 1;

   var variableIdentifier =
      CheckIfFirstInvocationAssignedToVariableAndReturnVariableIdentifier(
         originalInvocationsMatchingFixRequestInvocation);

   //.... MORE CODE
}
```

Let's also re-introduce the very simple example from [Part
2](https://blog.emirosmanoski.mk/2020-11-02-Roslyn-Roslyn-Analyzer-Part2/) that
is going to help us match some of the fix code with an actual use case. The
example is slightly modified to better match one of the checks that occurs during step 2). 

``` csharp
class Program
{
   static void Main(string[] args)
   {
      var input = 4;
      int y;

      int x = Foo(input);
      y = Foo(input);

      Console.WriteLine(x);
      Console.WriteLine(y);
   }

   static int Foo(int i)
   {
      return 4;
   }
}
```

We initially get the `expressionName` of the invocation for which our Analyzer discovered multiple calls with the same paramters. If we look at the simple example that would be the name of the method: `Foo`.

We then create the object that will help us make the changes to the code, the [`documentEditor`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.editing.documenteditor?view=roslyn-dotnet).

What we do next is find all invocations within the the current document that
match the expression by using the `OriginalRoot` on the `documentEditor` by
matching on the `expressionName`. If the count of such invocations is one we
don't really need to provide a fix.

The last line calls the
`CheckIfFirstInvocationAssignedToVariableAndReturnVariableIdentifier` function
and expects us to potentially receive the variable which we will use to perform the check in step 2).

### Check initial invocation variable

Let's have a look at the `CheckIfFirstInvocationAssignedToVariableAndReturnVariableIdentifier`:

``` csharp
 private static SyntaxToken CheckIfFirstInvocationAssignedToVariableAndReturnVariableIdentifier(
            List<InvocationExpressionSyntax> allInvocationExpressions)
{
   // we need the first invocation
   var firstInvocation = allInvocationExpressions.First();

   // get the top level "invocation" container
   var topLevelInvocationContainer = GetNodeTopLevelContainerThatIsChildOfCodeBlock(firstInvocation);

   // The variable identifier of the first element
   var variableIdentifier = new SyntaxToken();

   // check if it is a variable declaration meaning that we will have a variable we can re-use
   if (topLevelInvocationContainer.IsKind(SyntaxKind.LocalDeclarationStatement))
   {
         var variableDeclarator =
            topLevelInvocationContainer
               .DescendantNodes().ToList().Where(x => x.IsKind(SyntaxKind.VariableDeclarator))
               .FirstOrDefault() as VariableDeclaratorSyntax;

         variableIdentifier = variableDeclarator.Identifier;
   }

   return variableIdentifier;
}
```

The function is a simplified check that relies on our first method invocation
being part of a [`LocalDeclarationStatementSyntax`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntax.localdeclarationstatementsyntax?view=roslyn-dotnet) Syntax Node.

The change we made to our example was specifically to match the `LocalDeclarationStatementSyntax`. For a better view of what is going on let's have a look at the Syntax Visualizer to identify the full hierarchy for the first invocation of the Foo Method:

{:refdef: style="text-align: center;"}
![Foo Invocation Expression property in Tree](/images/2020-11-16-Roslyn-Syntax-Analyzer-Part-3/01_SyntaxVisualizer_InvocationExpression.png)
{:refdef}

What our method does is look for the `VariableIdentifier` from the `Variable Declarator Syntax` by looking at the top-most Syntax Node that is a child of a Code Block which contains our `Invocation Expression`.

If no such node exists - we return will return the IdentifierSyntax otherwise we return a default "empty" `SyntaxToken`.

The `GetNodeTopLevelContainerThatIsChildOfCodeBlock(firstInvocation)` method is very simple and just traverses the Invocation Ancestors finding the first node that is child of a Block Syntax Node. 

``` csharp
private static SyntaxNode GetNodeTopLevelContainerThatIsChildOfCodeBlock(SyntaxNode node)
{
   return node.FirstAncestorOrSelf<SyntaxNode>(x => x.Parent.IsKind(SyntaxKind.Block));
}
```

{: .box-warning}
**Note** This is one method that is simplified and relies on the `LocalDeclarationStatement` explicitly. This can be improved by including some additional checks and potentially semantic analysis that would work beyond just this simple example. For simplicity we modified the simple `Foo()` example to introduce the `LocalDeclarationStatement` as the first invocation to `Foo()`. For comparison in the example the invocation for `y` is a regular `ExpressionStatement` as we can see in the image: `ExpressionStatement[216..231]`. The identifier can be extracted from such expressions as well, but it would require a slightly different syntax tree traversal/search.
{: .box-warning}

At this point we have everything we need to perform our check in step 2)! :heavy_check_mark:

### Introducing a variable if first Invocation is not part of Local Declaration

Our previous code examples looked at how we find if the first invocation of the
method is assigned to a variable. We store that variable in the
variableIdentifier `SyntaxToken` and we kick off the next steps by checking it's
`SyntaxKind`.

If there the first invocation is not assigned to a variable the SyntaxKind of
the `variableIdentifier` will be None. If that is the case we proceed to create
our own variable to which we will assign the return value of the first
invocation.

``` csharp

var variableIdentifier =
      CheckIfFirstInvocationAssignedToVariableAndReturnVariableIdentifier(
         originalInvocationsMatchingFixRequestInvocation);

// ... PREVIOUS CODE
// this means the first invocation of multiple is not assigned to a variable
// we will have to create a new invocation and assign it to a variable we can use to replace
// all the other invocations
if (variableIdentifier.IsKind(SyntaxKind.None))
{
      var variableDeclarator = SyntaxFactory
         .VariableDeclarator(GetNewDeclaredVariableForExpression(expressionName))
         .WithInitializer(
            SyntaxFactory.EqualsValueClause(
                  invocationRequestingFix
            )
      );

      // Create the Variable declaration.
      var variableDeclaration = SyntaxFactory.VariableDeclaration(TypeSyntaxFactory.GetTypeSyntax("var"))
         .WithTrailingTrivia(SyntaxFactory.SyntaxTrivia(SyntaxKind.WhitespaceTrivia, " "))
         .WithVariables(SyntaxFactory.SeparatedList<VariableDeclaratorSyntax>().Add(variableDeclarator));

      var localDeclaration = SyntaxFactory.LocalDeclarationStatement(variableDeclaration)
         .WithAdditionalAnnotations(Formatter.Annotation);

      AddLocalDeclarationBeforeFirstReplacedInvocation(originalInvocationsMatchingFixRequestInvocation,
         localDeclaration,
         documentEditor);

      variableIdentifier = variableDeclarator.Identifier;
      originalInvocationsToSkipWhenFixing = 0;
}
// ... MORE CODE
```

{: .box-note}
**Note:**  For us to reach this code we have to modifiy the `Foo()` example and replace `int x = Foo(input);` with just `Foo(input);`
{: .box-note}


This is where we start using a very important class to help us build syntax: [`SyntaxFactory`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntaxfactory?view=roslyn-dotnet).

This is where the `SyntaxVisualizer` and the [Roslyn Quoter](http://roslynquoter.azurewebsites.net/) can help us structure how we build our code through using the SyntaxFactory.

We initially create a `variableDeclarator` variable of type [`VariableDeclaratorSyntax`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntax.variabledeclaratorsyntax?view=roslyn-dotnet) using a variable name created in the `GetNewDeclaredVariableForExpression` method and initializing the variable using [`EqualsValueClauseSyntax`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntax.equalsvalueclausesyntax?view=roslyn-dotnet) by setting the expression on the right side of the Equals Clause to the `InvocationExpression` we are trying to fix.

{: .box-note}
**Note:**  `GetNewDeclaredVariableForExpression` is a simple method that generates the new variable name. It contains code does some validation and sanitization of the expression name for certain cases and then tries to compose a name that follows the pattern: `{SANITIZED_FUNCTION_NAME}Result`. For the `Foo()` example we end up with  `fooResult` as the variable name as we will see.
{: .box-note}

The syntax that we've generated so far is:

``` csharp
fooResult = Foo(input)
```

And we can see that when debugging the extension:

{:refdef: style="text-align: center;"}
![Variable Declarator Syntax](/images/2020-11-16-Roslyn-Syntax-Analyzer-Part-3/02_VariableDeclaratorSyntax.png)
{:refdef}

This is still not fully correct C# syntax. We are missing either the explicit
`type` for `fooResut` or the `var` keyword as well as some additional syntax and
formatting. 

Now lets look at the following lines that generate the full new expression:

``` csharp
// Create the Variable declaration.
var variableDeclaration = SyntaxFactory.VariableDeclaration(TypeSyntaxFactory.GetTypeSyntax("var"))
   .WithTrailingTrivia(SyntaxFactory.SyntaxTrivia(SyntaxKind.WhitespaceTrivia, " "))
   .WithVariables(SyntaxFactory.SeparatedList<VariableDeclaratorSyntax>().Add(variableDeclarator));

var localDeclaration = SyntaxFactory.LocalDeclarationStatement(variableDeclaration)
   .WithAdditionalAnnotations(Formatter.Annotation);
```

The `variableDeclaration` uses the already defined `variableDeclarator` and adds
a `TypeSyntax` Node using the `var` keyword. Here we also specify whitespace
[`TrailingTrivia`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.syntaxtoken.trailingtrivia?view=roslyn-dotnet#Microsoft_CodeAnalysis_SyntaxToken_TrailingTrivia).
This adds the white space between the `var` and `fooResult = Foo(input)` syntax
constructs. Finally we define the variables we are declaring - in our case the
`fooResult` variable as specified with the `variableDeclarator`. 

At this point the syntax described by `variableDeclaration` is:

``` csharp
var fooResult = Foo(input)
```

or as it can be seen by the value during debug:

{:refdef: style="text-align: center;"}
![Variable Declarator Syntax](/images/2020-11-16-Roslyn-Syntax-Analyzer-Part-3/03_VariableDeclarationFull.png)
{:refdef}

This is still not a fully valid expression because of the missing `;` so we further define a [`localDeclaration` statement](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntax.localdeclarationstatementsyntax?view=roslyn-dotnet) to which we add additional annotations in terms of formatting. Finally this specifies the declaration as a valid expression:

``` csharp
var fooResult = Foo(input);
```

or as it can be seen during debug:

{:refdef: style="text-align: center;"}
![Variable Declarator Syntax](/images/2020-11-16-Roslyn-Syntax-Analyzer-Part-3/04_LocalDeclarationStatementpng.png)
{:refdef}

This gives us a statement that we can now insert into our current code which
calls the method and stores the results in a variable. We can then use the
variable as a replacement for the other calls in the block.

What's left to do now is add our `localDeclaration` statement in the code before the first invocation:

``` csharp
AddLocalDeclarationBeforeFirstReplacedInvocation(originalInvocationsMatchingFixRequestInvocation,
localDeclaration,
documentEditor);

variableIdentifier = variableDeclarator.Identifier;
originalInvocationsToSkipWhenFixing = 0;
```

Once we do that we now set the `variableIdentifier` object we inspected before
and state that when the next part of the code runs all of the invocations will
need to be replaced with our newly introduced variable.

{: .box-note}
**Note:**  Remember that we are looking at one of two branches in our code where the first invocation of the "offending" method/function is not assigned to a variable through a local declaration. This is code that would potentially not run if the first invocation result was stored in a locally declared variable. In that case `originalInvocationsToSkipWhenFixing` would be set to the initial value of 1, meaning no work in the following code would be done to the first invocation.
{: .box-note}

Let's have a look now at `AddLocalDeclarationBeforeFirstReplacedInvocation`:

``` csharp
private static void AddLocalDeclarationBeforeFirstReplacedInvocation(
            List<InvocationExpressionSyntax> allInvocationExpressions,
            LocalDeclarationStatementSyntax localDeclaration, DocumentEditor editor)
{
   var firstInvocationOccurrence = allInvocationExpressions.First();

   var firstInvocationTopLevelParent = GetNodeTopLevelContainerThatIsChildOfCodeBlock(firstInvocationOccurrence);

   var locDeclaration = localDeclaration.WithLeadingTrivia(firstInvocationTopLevelParent.GetLeadingTrivia());

   editor.InsertBefore(firstInvocationTopLevelParent, locDeclaration);
}
```

This is a relativized straightforward way of inserting code using the [`DocumentEditor.InsertBefore`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.editing.syntaxeditor.insertbefore?view=roslyn-dotnet#Microsoft_CodeAnalysis_Editing_SyntaxEditor_InsertBefore_Microsoft_CodeAnalysis_SyntaxNode_System_Collections_Generic_IEnumerable_Microsoft_CodeAnalysis_SyntaxNode__) method.

We first get the `firstInvocationOccurrence` of all the method invocations and use `GetNodeTopLevelContainerThatIsChildOfCodeBlock`. Remember that this returns the top most `expression` containing the `InvocationExpression`. We want to insert our `LocalDeclarationStatementSyntax` exactly before this expression.

To make sure we keep formatting and any custom changes (whitespace) that the
developer might have added we need to make sure we keep the `Leading Trivia`
which comes before the top level expression of the first invocation.

To do that we create a "new" `LocalDeclarationStatement` out of our original one
by using `WithLeadingTrivia` and passing along the Leading trivia of the
`firstInvocationTopLevelParent`. This makes sure the code structure remains the
same.At the same time `firstInvocationTopLevelParent` gets to keep its leading
trivia. 

Finally we use the editor and insert our new `locDeclaration` before the
`firstInvocationTopLevelParent`. With that we are done with defining/declaring
and making changes to the file.

### Variable Identifier Summary

Before moving on let's do e recap of where we are in terms of the original 5
steps. We are at the point where we can execute the last step which is going to
replace all of the invocations of a method with the variable defined with the
`variableIdentifier` syntax.

What `variableIdentifier` syntax contains is going to depend on the first
invocation of the method for which our diagnostic has been reported.

If the first invocation was in the general format of:

``` csharp
var output = MethodCall(input);
DoSomethingWithThisInput(output);
DoSomethingElseWithThisInput(MethodCall(input));
```

our `variableIdentifier` is going to point to `output`. If our first invocation was in the general format of:

``` csharp
DoSomethingWithThisInput(MethodCall(input));
DoSomethingElseWithThisInput(MethodCall(input));
```

we do some pre-processing and end up initially with:

``` csharp
var methodCallResult = MethodCall(input);
DoSomethingWithThisInput(MethodCall(input));
DoSomethingElseWithThisInput(MethodCall(input));
```

In the next step (5) we will see how we take the `variableIdentifier` and replace all valid instances of `MethodCall(input)` so we  end up the least amount of calls and re-use the variable as much as possible. Depending on which scenario we are dealing out of all of the Invocation Instances we might replace either all, or skip the first invocation. For example for `var output = MethodCall(input);` we skip the first invocation.

### Replacing Invocations with Variable Identifier

The final part of our `CodeAction` is as follows:

``` csharp
var invocationsToReplace =
   originalInvocationsMatchingFixRequestInvocation.Skip(originalInvocationsToSkipWhenFixing).ToList();

foreach (var invocation in invocationsToReplace)
{
   var invocationParent = invocation.Parent;

   var newNode = SyntaxFactory.IdentifierName(variableIdentifier.Text);
   var newInvocationParent = invocationParent.ReplaceNode(invocation, newNode);

   documentEditor.ReplaceNode(invocationParent, newInvocationParent);
}

return documentEditor.GetChangedDocument();
```

We start of by defining `invocationsToReplace` from the `originalInvocationsMatchingFixRequestInvocation` collection. We skip `originalInvocationsToSkipWhenFixing` which is either going to be 1 or 0 depending on the use cases described in the previous section. 

We then iterate over the invocations and to the following:

1. We find the invocation **Parent Node** which can be different types of `Syntax Nodes`
2. We create a new [`IdentifierNameSyntax`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.csharp.syntax.identifiernamesyntax?view=roslyn-dotnet) node from our `variableIdentifier` 
3. We "create" a  **New Parent Node** by replacing its _child_ `invocation` node with our new `IdentifierName` node using the `ReplaceNode` method.
4. We use the `documentEditor` to replace the original `invocationParent` which contains the `invocation` with the `newInvocationParent` which contains the replaced `newNode` (the IdentifierName).

{: .box-note}
**Note:**  The `SyntaxNode` methods like `ReplaceNode` are immutable. They don't modify the node on which we call them. They create new `Syntax Nodes` based on the changes.
{: .box-note}

As a final step we call and return [`documentEditor.GetChangedDocument();`](https://docs.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.editing.documenteditor.getchangeddocument?view=roslyn-dotnet) which applies our changes to the existing document.

With this our `Code Action` takes effect and we've fully executed our code fix! :heavy_check_mark:.

{: .box-warning}
**Warning:**  This final part of the code is simplified and is only for example purposes. The replacements don't take into account that potentially some of the invocations  might be using different input parameters. The fix provider should either share analysis code with the analyzer to check for parameters or a different approach potentially using semantic checks can be taken to know exactly which methods to replace.
{: .box-warning}

## In Action

Using our `Foo()` method example:

``` csharp
class Program
{
   static void Main(string[] args)
   {
      var input = 4;
      int y;

      int x = Foo(input);
      y = Foo(input);

      Console.WriteLine(x);
      Console.WriteLine(y);
   }

   static int Foo(int i)
   {
      return 4;
   }
}
```

We will see that if we "focus" on our `Fix` as the solution to the squiggly lines reported by our [Analyzer](https://blog.emirosmanoski.mk/2020-11-02-Roslyn-Roslyn-Analyzer-Part2/) the IDE shows a preview of the code changes:

{:refdef: style="text-align: center;"}
![Foo Invocation Expression property in Tree](/images/2020-11-16-Roslyn-Syntax-Analyzer-Part-3/05_FixPreview.png)
{:refdef}

Applying the fix results in the fixed code with the second invocation being removed and replaced by the value stored in the variable `x`

``` csharp
class Program
{
   static void Main(string[] args)
   {
      var input = 4;
      int y;

      int x = Foo(input);
      y = x;

      Console.WriteLine(x);
      Console.WriteLine(y);
   }

   static int Foo(int i)
   {
      return 4;
   }
}
```

## Final Notes

The full version of the Fix Provider can be found at the following
[LINK](https://gist.github.com/emir01/ee980807fed54736068118e41c84f7ba).

And with that we finish the series on Roslyn. As we can see Roslyn (.NET
Compiler Platform SDK) provides us a set of quite powerful tools we can utilize
to improve the overall coding experience and the quality of our code.

What we saw in this series is just barely scratching the surface of what can be
done and as mentioned some of our code here makes certain assumptions and
certain simplifications for example purposes.

Nevertheless it can serve as an interesting introduction to the Platform and it
was definitely an interesting experience to write. :satisfied: :fireworks:

For a more in-depth set of learning resources for Roslyn I'd recommend looking at the following
series:

- [Learn Roslyn Now](https://joshvarty.com/learn-roslyn-now/)
