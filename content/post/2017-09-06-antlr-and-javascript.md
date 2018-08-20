---
title: 'ANTLR and JavaScript'
date: '2017-09-06'
categories:
- crystalgazer
tags:
- javascript
- antlr
comments: []
---

In the last few weeks I've been working on Crystal Gazer, a nodejs console application to gather information from your Git repository. You can find it on [GitHub](https://github.com/vgaltes/CrystalGazer) and on [NPM](https://www.npmjs.com/package/crystalgazer).

One of the things I'd like to do is to track the evolution of a function. Has it been modified a lot? How many people has been working on it? Is the function too long? 

To answer the first question, what we could do is rely on the *git log -L:function:file* command to give us all the changes a function has suffered. We need a couple of things to run this command: the name of the file and the name of the function.

The name of the file is the easy part. We have the names of the files in the original git log (see the [README.md](https://github.com/vgaltes/CrystalGazer/blob/master/README.md) of Crytal Gazer to know how it works) and we can always ask the user to input it. But given a file, we'd like to list the different functions with their statistics. So we need an automated way to extract the name of the functions from a source code file (we're going to work with .cs files).

## First attempt

My first approximation was to try to code something by myself. A combination of Regex, a couple of ifs, maybe some split... That didn't worked well. The number of cases to take into account is big enough to be very difficult to write such function.

## ANTLR

Then, one acronym came to my mind: AST. An Abstract Syntax Tree is a tree representation of the abstract syntactic structure of source code written in a programming language ([wikipedia](https://en.wikipedia.org/wiki/Abstract_syntax_tree)). This is what, for example, [escomplex](https://github.com/escomplex/escomplex) uses to perform metrics. Can we create the AST of C# code in JavaScript?

The answer is [ANTLR](http://www.antlr.org/). ANTLR is a parser generator that can create an AST from a file using a grammar that we can define. Fortunately, we don't need to create the C# grammar by ourselves, we can grab it from [here](https://github.com/antlr/grammars-v4).

These are the steps I followed to be able to get the function names from a C# file.

### Create the lexer, parser and the listener from the grammar.
In a **VERY** short way, antlr uses a lexer to create tokens from your input, then uses these tokens to initialize the parser that creates the AST. When you traverse the tree, you get notified in the listener about the different things it finds.

To create these three files, we'll need to use the antlr tool. So, let's [download](http://www.antlr.org/download/antlr-4.7-complete.jar) the tool and make it accessible. Follow these [instructions](https://github.com/antlr/antlr4/blob/master/doc/getting-started.md) just changing 4.5.3 to 4.7. 

Now we can use the tool to generate the lexer. Download CSharpLexer.g4 from the [grammars repository](https://github.com/antlr/grammars-v4/tree/master/csharp) and copy it to your project folder. Run the following command:
```
antlr4 -Dlanguage=JavaScript CSharpLexer.g4
```

This will generate CSharpLexer.js and CSharpLexer.tokens.

We need to do the same for [CSharpParser.g4](https://github.com/antlr/grammars-v4/blob/master/csharp/CSharpParser.g4). So run the following command:

```
antlr4 -Dlanguage=JavaScript CSharpParser.g4
```

In this case this command will generate CSharpParser.js, CSharpParser.tokens and CSharpParserListener.js.

### Prepare a nodejs project
We're going to create a nodejs project to use all this stuff. Let's add antlr4 to it:

```
npm install antlr4 --save
```

Create a file called index.js and import the neccessary modules:

```
const antlr4 = require('antlr4/index');
const fs = require('fs');
const CSharpParser = require('./CSharpParser.js');
const CSharpLexer = require('./CSharpLexer.js');
```

If you now run this script you'll get errors loading the CSharpLexer module. That's because the file generated has some invalid identifiers:
 - In the line 5 there's a `import java.util.Stack;` which is obviously invalid in JavaScript. Just comment out (or delete) the line.
 - In the lines 1770 to 1773 (both included) there is some C# code. Just change each variable initialisation by `var <variable_name>`.
 - Between the lines 1866 and 1883 (both included) there is more C# code. Comment out or delete it.

### Creating the listener
With the help of ANTLR we've created a CSHarpListener. We can use that listener as a *base class* for more specific listeners. As we want to get the names of the different methods of the file, let's create a listener for that.

Create a file called CSharpFunctionListener.js and copy the following code:

```
const antlr4 = require('antlr4/index');
const CSharpLexer = require('./CSharpLexer');
const CSharpParser = require('./CSharpParser');
var CSharpListener = require('./CSharpParserListener').CSharpParserListener;

CSharpFunctionListener = function(res) {
    this.Res = res;    
    CSharpListener.call(this); // inherit default listener
    return this;
};
 
// inherit default listener
CSharpFunctionListener.prototype = Object.create(CSharpListener.prototype);
CSharpFunctionListener.prototype.constructor = CSharpFunctionListener;


CSharpFunctionListener.prototype.enterMethod_member_name = function(ctx){
    this.Res.push(ctx.getText());
}

exports.CSharpFunctionListener = CSharpFunctionListener;
```

Nothing too special here. The important part is that we're overriding the *enterMethod_member_name* method, which is the method that will be called when the parser finds a method name.

### Putting all together

Time to use all this stuff. Go back to your index.js file and add the following code:

```
var input = fs.readFileSync('aFile.cs').toString();

var chars = new antlr4.InputStream(input);
var lexer = new CSharpLexer.CSharpLexer(chars);
var tokens  = new antlr4.CommonTokenStream(lexer);
var parser = new CSharpParser.CSharpParser(tokens);

var tree = parser.namespace_member_declarations();   
var res = [];
var csharpClass = new CSharpFunctionListener(res);
antlr4.tree.ParseTreeWalker.DEFAULT.walk(csharpClass, tree);

console.log("Function names: ", res.join(','));
```

As you can see, we're creating the lexer and the parser. Then, we use antlr to traverse the tree and pass the listener in order to be notified. 

If you run it, you will get the names of the functions.

## Summary
This is the first thing I do with antlr and it's been really pleasant. The only problem was removing the invalid code from the lexer and that's all. Expect a new version of Crystal Gazer using some antlr magic soon!!