+++
title = "How I Fixed Roslyn"
date = "2016-10-31T14:40:42-04:00"
draft = false

+++

After hearing so many good things about VS 15 I wanted to give it a test run on some of our production code. Visual Studio was great, but ran into some interesting issues while executing some of our unit tests. Everything was compiling fine, but on a handful of tests I'd get the following runtime error

``` output
Unhandled Exception: System.ArgumentException: Expression of type 'System.Int32' cannot be used for return type 'System.Nullable`1[System.Int32]'
   at System.Linq.Expressions.Expression.ValidateLambdaArgs(Type delegateType, Expression& body, ReadOnlyCollection`1 parameters)
   at System.Linq.Expressions.Expression.Lambda[TDelegate](Expression body, String name, Boolean tailCall, IEnumerable`1 parameters)
   at System.Linq.Expressions.Expression.Lambda[TDelegate](Expression body, Boolean tailCall, IEnumerable`1 parameters)
   at System.Linq.Expressions.Expression.Lambda[TDelegate](Expression body, ParameterExpression[] parameters)
   at ....
```

At first I didn't know what to make of it. Frustratingly enough the majority of the code that this was occuring in was relatively complicated join statements in EF or part of a complex validation rule that we set up with FluentValidation. I could easily resolve the issue by with an explicit cast or even moving the expression to an explicitly defined variable. But that would only fix the code I knew I had unit tests around and we are far from 100% code coverage. So I wanted to get to the heart of the issue.

Step one was trying to figure out how to reproduce it, but without code using a third party library. Not going to lie - this was total guess and check. The FluentValidation code was tightly wound to the external library so it was right out, but the EF queries I felt was a good start. I could maybe just mimic the join that was failing using a couple of `List<>` and be good to go. I knew it had to do with casting a nullable int to a regular int, so the simplest example I could come up with that still showed the error was 

``` csharp
class Program
{
    static void Main(string[] args)
    {
        var invoices = new List<Invoice> { new Invoice { InvoiceId = 0 } };
        var oneTimeCharges = new List<OneTimeCharge> { new OneTimeCharge { Invoice = 0, OneTimeChargeId = 0 } };
        var otcCharges = invoices.Join(oneTimeCharges, inv => inv.InvoiceId, otc => otc.Invoice, (inv, otc) => inv.InvoiceId);
        Console.WriteLine(otcCharges.Count());
    }        
}

public class OneTimeCharge
{
    public int OneTimeChargeId { get; set; }
    public int? Invoice { get; set; }
}

public class Invoice
{
    public int InvoiceId { get; set; }
}
```

Threw this into a console application, hit run aaaand....it worked. No errror. I had already reproduced the issue using FluentValidation so I knew the error could be trigged from this console app. I took the dog for a walk and started thinking that maybe EF wasn't causing it but rather it being of type `IQueryable<>` caused the lambdas to be compiled differently. I headed back inside and threw an `AsQueryable()` onto the end of both definitions and hit run and....jackpot. Got the failure message and I was never happier to see some code fail.

Elated that I had a reproducable issue I naturally wanted to verify why it was happening. Luckily there's a great little tool at http://tryroslyn.azurewebsites.net/. I was able to copy and paste my little app here and check out the code that's generated using both the version in VS 15 (master) and the current nuget release and compare the difference. 

Armed with this I found the code at fault. In the current version the following code is produced:

``` csharp
Expression<Func<Invoice, int?>> arg_EE_2 = Expression.Lambda<Func<Invoice, int?>>(Expression.Convert(Expression.Property(parameterExpression, methodof(Invoice.get_InvoiceId())), typeof(int?)), new ParameterExpression[]
{
    parameterExpression
});
```

In the version in VS 15 (master) this was produced:

``` csharp
Expression<Func<Invoice, int?>> arg_DF_2 = Expression.Lambda<Func<Invoice, int?>>(Expression.Property(parameterExpression, methodof(Invoice.get_InvoiceId())), new ParameterExpression[]
{
    parameterExpression
});
```

Note the missing Expression.Convert. Feeling good about myself for finding a reproducable test case headed over to Roslyn's Github repository to [file my bug](https://github.com/dotnet/roslyn/issues/14722). 

A few days passed and with my wife working night shifts I got the itch to see if I couldn't fix this myself. Heck, I got a C+ in compiler construction in college - I surely could do this.

Step one was getting Roslyn up and running locally. This was easy enough, just follow the instructions on the [Building Testing and Debugging
](https://github.com/dotnet/roslyn/wiki/Building%20Testing%20and%20Debugging) page. I walked through the instructions here to get the code building, and just to verify I did it right I decided to go the next step and run the unit test suite. 10 minutes later...success. But oomph that was a brutal wait, and I feared my current strategy of a guess and check bug fix would be a long road to hoe.

Still, I now had the code running locally so it was time to try and track down precisely when things broke. Hopefully with that information I could see what changed and figure out how to get it back to a working state. A quick look at the difference between what's in VS 2015 Update 3 and VS15 unfortately provide slightly overwhelming.

![3,700 commits](/img/blog/how-i-fixed-roslyn/commits.png)

But I did have a powerful thing at my disposal. My reproducable test case was an actual program. I can automate this. Git has a nice tool called [`bisect`](https://git-scm.com/docs/git-bisect) just for this. TLDR of `git bisect` if you haven't used it - you give it a starting and ending spot of what you consider good and bad commits plus a command to run that returns an error code of 0 or 1 depending on if it passes. It then runs a binary search through the commits executing the command and moving on. I'd have to compile roslyn, but that's not a problem. Their build script is easy enough, and then I'd just have to compile the my test case, run it and check the error code. Thanks to the binary search of the bisect command instead of 3,700 commits I'd only need like 12 or 13 runs. Nice!

This did involve some trial and error. It seems the output of `csc.exe` changed as part of the changes. No problem, nothing a little batch file `if` statement couldn't resolve. `Git bisect` also wasn't a huge fan of the project.json files being modified in the build process so I also made sure to reset things before running my app so it could do its thing cleanly.  My script ultimately ended up looking like this:

``` dos
call Restore.cmd
msbuild /v:m /m /p:Configuration=Release Roslyn.sln

IF EXIST Binaries\Release\Exes\csc\csc.exe (
Binaries\Release\Exes\csc\csc.exe expression.cs
) ELSE (
Binaries\Release\csc.exe expression.cs
)

git reset --hard
expression.exe
```   

Expression.cs was my source and expression.exe was the compiled app. Time to launch the bisect. To do this I ran the following commands while in the master branch, which was current failing

``` dos
git checkout master
git bisect start
git bisect bad
git bisect good update3
git bisect run doit.cmd
```

I went to go refresh my drink and walk the dog. Ten minutes later I returned to the following message

``` console

6c9e18649f576bd9df1e0db8ad21bfbce0454704 is the first bad commit
commit 6c9e18649f576bd9df1e0db8ad21bfbce0454704
Author: Charles Stoner <chucks@microsoft.com>
Date:   Tue Jul 12 13:51:14 2016 -0700

    Support async methods returning Task-like types

    [Based on features/async-return with tests ported from that branch.]

:040000 040000 5be0af3e25b7c68022d128a9adf5984e2eb03df3 d4ffb41d83a385fda5a1d60e04d488cf081d1a2f M      src
bisect run success
```

Fantastic. Looks like I found the guilty commit. Unfortunately, well, this was a huge commit. 

![Lots of changes](/img/blog/how-i-fixed-roslyn/huge-commit.png)

Sigh. One thing to do at this point - `CTRL-F` and search for lambda in the [commit on github](https://github.com/dotnet/roslyn/commit/6c9e18649f576bd9df1e0db8ad21bfbce0454704). Eye balling the highlighted results showed a couple of hot spots with `lambda` in the code frequently so I quickly scrolled there. After a few seconds of panic of this being a needle in a haystack I happened to glance down and see this 

![Is that a debug.assert](/img/blog/how-i-fixed-roslyn/lambda-assert.png)

That looks like somewhere to start! But I needed a better way to test than hitting the command line and compiling over and over again. I needed a unit test. I found a file named [CodeGenExprLambdaTests](https://github.com/dotnet/roslyn/blob/master/src/Compilers/CSharp/Test/Emit/CodeGen/CodeGenExprLambdaTests.cs) and figured this was as good as place as any to write my test. Opening this up I discovered a suite of tests that were verifying compiler output and executing the results. Perfect! All I had to do was mimic their style and hopefully I would be in business. Even better I noticed Resharper picked up the tests so just maybe I could use Resharper to only run the test I was working on. In retrospect I probably should have started here, but oh well.

Thanks to a little copy and paste development it looked like this is the test I wanted to run. I could copy my full app here and they have a nice way to run the application and verify the output is correct.

``` csharp
[Fact]
public void ConversionAppliedInLambdaForNonMatchingTypes()
{
    var program = @"
using System;
using System.Collections.Generic;
using System.Linq;
namespace ConsoleApplication2
{
    class Program
    {
        static void Main(string[] args)
        {
            var invoices = new List<Invoice>().AsQueryable();
            var oneTimeCharges = new List<OneTimeCharge>().AsQueryable();
            var otcCharges = invoices.Join(oneTimeCharges, inv => inv.InvoiceId, otc => otc.Invoice, (inv, otc) => inv.InvoiceId);
            Console.Write('k');
        }        
    }

    public class OneTimeCharge
    {
        public int OneTimeChargeId { get; set; }
        public int? Invoice { get; set; }
    }
    public class Invoice
    {
        public int InvoiceId { get; set; }
    }    
}
";

    CompileAndVerify(
        sources: new string[] { program, ExpressionTestLibrary },
        additionalRefs: new[] { ExpressionAssemblyRef },
        expectedOutput: @"k")
        .VerifyDiagnostics();
}
```

I ran the unit test using R# with optimism, and instead of seeing an exception about casting I actually hit the `Debug.Assert`. Nice!

``` console
---------------------------
Assertion Failed: Abort=Quit, Retry=Debug, Ignore=Continue
---------------------------
   at Microsoft.CodeAnalysis.CSharp.UnboundLambdaState.ReallyBind(NamedTypeSymbol delegateType) in C:\Projects\phil-roslyn\src\Compilers\CSharp\Portable\BoundTree\UnboundLambda.cs:line 433
   at Microsoft.CodeAnalysis.CSharp.UnboundLambdaState.Bind(NamedTypeSymbol delegateType) in C:\Projects\phil-roslyn\src\Compilers\CSharp\Portable\BoundTree\UnboundLambda.cs:line 345
   at Microsoft.CodeAnalysis.CSharp.UnboundLambda.Bind(NamedTypeSymbol delegateType) in C:\Projects\phil-roslyn\src\Compilers\CSharp\Portable\BoundTree\UnboundLambda.cs:line 277
   at Microsoft.CodeAnalysis.CSharp.Binder.CreateAnonymousFunctionConversion(SyntaxNode syntax, BoundExpression source, Conversion conversion, Boolean isCast, TypeSymbol destination, DiagnosticBag diagnostics) in C:\Projects\phil-roslyn\src\Compilers\CSharp\Portable\Binder\Binder_Conversions.cs:line 300
   at Microsoft.CodeAnalysis.CSharp.Binder.CreateConversion(SyntaxNode syntax, BoundExpression source, Conversion conversion, Boolean isCast, Boolean wasCompi......

<truncated>
---------------------------
Abort   Retry   Ignore   
---------------------------
```

I had the [line and everything to start with](https://github.com/dotnet/roslyn/blob/aa52fe081859f640e461a767cbf195bbca64a58d/src/Compilers/CSharp/Portable/BoundTree/UnboundLambda.cs#L433). I quickly edited my unit test to a passing version without AsQueryable and ran it. Success! Good news all around. Not only does the unit test runner work, but I had a great starting spot for what is going wrong.

The line of code in question that was failing was this section of code.

``` csharp
if (_returnInferenceCache.TryGetValue(cacheKey, out returnInferenceLambda) && returnInferenceLambda.InferredFromSingleType)
{
    lambdaSymbol = returnInferenceLambda.Symbol;
    Debug.Assert(lambdaSymbol.ReturnType == returnType);
    Debug.Assert(lambdaSymbol.RefKind == refKind);

    lambdaBodyBinder = returnInferenceLambda.Binder;
    block = returnInferenceLambda.Body;
    diagnostics.AddRange(returnInferenceLambda.Diagnostics);

    goto haveLambdaBodyAndBinders;
}

```

Right there is the Debug.Assert failing when ReturnType doesn't equal whatever this lamdaSymbol.Return type is. Huh. So it looks like they are caching previously generated expressions if they match, but it appears we are somehow pulling one out that doesn't have a matching return type. Running the test through the debugger everything else seemed to jive, just not this. Looking at the end of the if statement that was already checking `InferredFromSingleType` it appeared to me that maybe more filtering was needed here. So I added one more check

``` csharp
if (_returnInferenceCache.TryGetValue(cacheKey, out returnInferenceLambda) && returnInferenceLambda.InferredFromSingleType && returnInferenceLambda.Symbol.ReturnType == returnType)
...
```

Reran my test and...success! Happy days. 

Armed with a reproducable unit test and a fix I mustered up the courage to [submit a PR](https://github.com/dotnet/roslyn/pull/14755). After receiving some feedback I was pumped to see it accepted and merged into the codebase in time for the VS 15 RC release hopefully.

Postscript
----------

I never felt like I had gotten this implementation correct. Kind of felt like things were either being put into or pulled out of cache improperly, and I was just filtering those out. But to be honest I didn't have enough confidence to go digging in and changing things here. But a few days later I noticed a new issue pop up sparked by my PR [new issue](https://github.com/dotnet/roslyn/issues/14774) around this.  Looks like as I suspected my fix is just a bandaid, and the caching is in fact the true bug. I fully expect this fix will cause my little fix to go away, but that's ok. Hopefully my test will remain to verify the behavior, and even if my tiny bit of code isn't running in the compiler I'm still pretty happy to be able to contribute to something like Roslyn.   