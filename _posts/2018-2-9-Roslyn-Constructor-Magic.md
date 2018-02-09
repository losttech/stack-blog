---
layout: post
title: Roslyn Constructor Magic
categories: roslyn AST
excerpt_separator: <!--more--> 
---

Mostly, we are doing a line to line translation. E.g. every line in Python code roughly translates into a line of C#, preserving order. However, there are differences between Python and C# semantics, that, unfortunately, force SharPy to perform more complicated transformations.

For example, implementations of ```__init__``` method may call parent class ```__init__``` anywhere in the body. This is not so for C# and base constructor calls, which have a form of

```csharp
ClassName(): base(...){...}
```

As you can see, the base constructor call is separeted from the derived class constructor body with a special syntax, and has to be called first. That means one must make a potentially complicated syntax tree transform to convert one to another. Fortunately, C# compiler API codenamed Roslyn was made exactly to do this, and is here to rescue.

<!--more-->

Let's start with what we've got.

```python
class Base:
  pass

class Derived(Base):
    def __init__(self):
        super().__init__()
        self.field = 1
```

Before we started working on constructor translation, this is what SharPy would output (shortened):

```csharp
class Base{}
class Derived: Base {
  public int field;
  public Derived() {
    base();
    this.field = 1;
  }
}
```

This is not a valid C#! Unlike Java, you can't just call base class constructor in the middle of your own constructor. It has to be written like in the example at the beginning of the post!

At this moment, you basically have 3 options:
1. extend Python AST library to support C#-style base calls and transform Python AST
1. make an intermediate AST, that incorporates features of both languages, and work with it
1. have C# AST support Python-style base calls

Intermediate AST is always a lot of work and code. Adding C# constructs to Python AST library is bad, because it would violate single responsibility principle.

Fortunately, Roslyn works perfectly well <span class="sidenote" title="unless you need semantic analysis OFC">even with a semantically invalid C# AST</span>!

As you can see in the old SharPy sample output, ```super().__init__()``` simply translates to ```base()```, which is not, however, placed correctly. Well, apparently, it is super easy to move it with Roslyn. Here's what I came up with:

```csharp
ConstructorInitializerSyntax ProcessBaseCall(ref BlockSyntax constructorBody) {
  if (constructorBody.Statements.FirstOrDefault() is ExpressionStatementSyntax statement
    && statement.Expression is InvocationExpressionSyntax invocation
    && invocation.Expression is BaseExpressionSyntax)
  {
    // if we are here, it means the first statement in the constructor body is the base constructor call
    // the call expression itself is placed into `invocation` variable
    constructorBody = constructorBody.WithStatements(constructorBody.Statements.RemoveAt(0));
    return ConstructorInitializer(SyntaxKind.BaseConstructorInitializer, invocation.ArgumentList);
  }
  return null;
}
```

At this moment I said to myself - "Wow!". I've been thinking on how to perform the translation in the spare time now and then, and it seemed like <span class="sidenode" title="And it is! You'll see below">a really complex issue</span>. Seeing the most common case implemented in 6 actual lines of simple code impressed me a lot.

Here's what this code does (in case its not clear on its own, otherwise - skip para): it takes in a constructor body block (```{ ... }```), and optionally returns ```ConstructorInitializerSyntax```, if the call to ```base(...)``` is the first statement in that constructor. ```ConstructorInitializerSyntax``` is basically the ```: base(...)``` part of C# syntax. So in this simple and very common scenario, ```ProcessBaseCall``` removes the unsupported statement from the broken body. It then returns ```: base(...)``` piece, so that caller would be able to inject it into the final definition.

That's it! We did not even have to reconstruct the argument list from scratch, since the original base call already had all the arguments in the correct order! So this code will automagically handle base constructors with parameters and/or different set of parameters from the base class.

Here's what we get now:

```csharp
class Base{}
class Derived: Base {
  public int field;
  public Derived():base() {
    this.field = 1;
  }
}
```

Cool, right?

Now scrupulous C# readers will notice, that it is not necessary to call a parameterless base constructor, and this code could be simplified by removing the whole ```:base()``` thing. Easy! Just check, that the original base call had any arguments:

```csharp
return invocation.ArgumentList.Arguments.Count > 0
  ? ConstructorInitializer(SyntaxKind.BaseConstructorInitializer, invocation.ArgumentList)
  : null;
```
And the full code now is:

```csharp
ConstructorInitializerSyntax ProcessBaseCall(ref BlockSyntax constructorBody) {
  if (constructorBody.Statements.FirstOrDefault() is ExpressionStatementSyntax statement
    && statement.Expression is InvocationExpressionSyntax invocation
    && invocation.Expression is BaseExpressionSyntax)
  {
    // if we are here, it means the first statement in the constructor body is the base constructor call
    // the call expression itself is placed into `invocation` variable
    constructorBody = constructorBody.WithStatements(constructorBody.Statements.RemoveAt(0));
    return invocation.ArgumentList.Arguments.Count > 0
      ? ConstructorInitializer(SyntaxKind.BaseConstructorInitializer, invocation.ArgumentList)
      : null;
  }
  return null;
}
```

This finally gives us perfectly valid and otherwise nice

```csharp
class Base{}
class Derived: Base {
  public int field;
  public Derived() {
    this.field = 1;
  }
}
```

Of course this is not the end of it. There's a more complex scenario yet to handle: since Python's constructor call potentially  can be anywhere in the body, somehow the code before it has to be injected into the ```: base(...)``` call to preserve semantics. Usually, that is done by introducing a static method(s) and calling them inside like this: ```Derived(): base(StaticMethod(...), ...)```. Or by having a factory method on the class. But this is a story for another post.