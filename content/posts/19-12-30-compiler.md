---
title: "Writing a Compiler: Crab"
date: 2019-12-20T4:35:00
slug: "crab-compiler"
draft: true 
tags: ["compilers", "projects"]
categories: ["projects"]
---

The purpose of this blog is to write about some of the various projects I've done as I've progressed in my learning. It is mostly for myself so that I can remember all the challenges I have faced and have some documentation of the things I've done. So let's get started. 

# Compilers
I like challenges. I like problems. And most importantly, I like understanding. As a programmer, I didn't understand how my code went from something in a text file to something that did stuff. So one of my first big projects, and the project that I am most proud of to this day, was writing a compiler from scratch. I decided to use OCaml, as it was the language I was learning at the time and there were many, many resources for writing a compiler in OCaml. 

I decided to write a small, end-to-end compiler for a language I made up called Crab. At the time I was in the summer of 7th grade, leading to 8th, and I wanted to undertake a fun project that I promised myself I would finish in its entirety. It took me almost 4 or 5 months during which I was learning and understanding and then writing code. I look back at Crab now, in my Junior year of high school, and there are some things that I think I did well for an 8th grader and other things that I want to change if I had the free time to actually go back and do so. To help ease my development, I used two libraries to assist me: LLVM and Menhir. 

# The stages of a compiler
Again, this post is mostly for myself, and I wrote the Crab compiler a long time ago, so I won't be going into too many code examples.

A compiler has many different stages, some books like to say that a compiler is just many tiny compilers all chained together. The idea is that you have some type of input, in this case, a file with Crab code, and it is run through a program that outputs it in a different state, in this case an executable. Within the compiler, the code goes from stage to stage, being "compiled" from one stage to another. 

### Lexing
For Crab, I used a built-in OCaml tool called OCamllex to do what is known as lexing. The code for that section of the compiler can be found [here](https://github.com/Heasummn/Crab-ML/blob/master/src/CrabParsing/lexer.mll).

Lexing is very interesting and no fine-grained explanation I give can do it proper justice, so I highly recommend you read a book or longer article focused on compilers or even just lexing. But at a macro-scale, a lexer uses some algorithm, of which there are many, to turn a file with text into an array of tokens. Each token is a representation of something in Crab: a number, special keywords, anything that is legal in the language. The lexer doesn't care about the order, it just makes sure that everything there is a part of the language. 

### Parsing
Once you've got this array of tokens, you want to take it and try to understand it. Parsing is probably the hardest and most complex stage of any simple compiler and just as with lexing, I cannot do it proper justice. Parsing takes this array of tokens and turns it into a tree, known as a parse tree. Each node represents a syntactic structure, like a function or an if statement, and inside the node are the other syntactic structures that make it up. For example, an if statement has the conditional statement that is inside of it and it has the body. A variable assignment has a name, a type, and a value. Ideally, once we're done parsing, the original syntax doesn't matter, and we could go back and make syntax changes that only affect the parser. 

All of these get read and parsed into a tree that is known as an Abstract Syntax Tree (AST). If there are any syntax errors in the code, the code halts and spits the error out. One of the main roadblocks I faced, and ignored, while building Crab was the fact that if the parser found an error, picking back up and continuing is very difficult. When you have a syntax error on a line, in say Java, the compiler will tell you about syntax errors on other lines as well. Once it finds the first error, it tries to find the next "correct" section and continue parsing. This was, and still is, a little over my head, and so Crab only reports the first syntax error it finds and quits. I would love to hear about any articles and papers that go into this idea because I couldn't find many when I was writing Crab. 

The parsing is done using a library called Menhir, that code is found [here](https://github.com/Heasummn/Crab-ML/blob/master/src/CrabParsing/parser.mly), and the AST is defined [here](https://github.com/Heasummn/Crab-ML/blob/master/src/CrabAst/crabAst.ml). 

One of the other problems I faced is fairly common in a statically typed language like OCaml. Each section of the AST looks like this:

```ocaml
type 'a annotation = { data: 'a; position: Location.t; tp: Types.tp }

type literal = simple_literal annotation
and simple_literal = 
    | Integer of int
    | Float of float
    | Bool of bool
```

I wanted to store information about what type each expression is since Crab is statically typed and we want to check the user's types. I also wanted information about where in the file and where in the line each expression is so that when I was reporting errors I could reference them. However, I didn't want to fill these in with the AST, I wanted them to be filled in during their respective stages. So, I had to create an intermediary, the parse tree. One of the mistakes that I made and something you should avoid doing, is the reuse of code. Each parse tree element looks like this:
```ocaml
type 'a annotation = { data: 'a; position: Location.t; }

type literal = simple_literal annotation
and simple_literal = 
    | Integer of int
    | Float of float
    | Bool of bool
```
They're identical, besides the type, and that is bad practice. If you're unfamiliar with OCaml, what I'm doing here is very similar, but not quite equivalent, to templates from other languages. I wrote Crab almost 4 years ago, and in retrospect, I would've just made everything have no type at first and gone straight to the AST, but I was young and dumb. 

### Intermediate Stages: Typing, Optimizations, Alpha Conversion
Once a compiler has the code in a parse tree, you can do a lot with it. I like to call each additional stage a "pass" over the tree, and the code for each pass that I have in the Crab language is defined in the [`CrabPasses`](https://github.com/Heasummn/Crab-ML/tree/master/src/CrabPasses) folder. 

The first thing I did was type everything and convert it from the parse tree into an AST. There are some cool things you can do with type inference, such as Hindley-Milner, that allow you to infer types of functions and arguments from how they are used, without needing the user to give them to you. This is what many functional languages like OCaml and Haskell do. Crab has none of those, although if I were to restart Crab development, I would try to implement type inference. Instead, everything is typed, and you can work from the bottom up to figure out what the type of each statement is and confirm that what the user gave matches.

I also didn't do any optimizations with the compiler, I instead relied on LLVM to do them for me, again something that I'd go back and change if I rebooted Crab. But the passes are the perfect place to do optimizations. For example, a very simple optimization would be to simplify any constant math, like `let n: int = 3 * 20`, and turn that into `let n: int = 60`. 

Finally, Alpha Conversion is pretty simple and required for any modern language. Crab allows functions with different types as input to exist under the same name. For example, a function named `parse` that takes a string, or a different function named `parse` that takes a list of strings (although Crab does not have support for strings). This is allowed in a language like C or Java, but assembly and machine code don't let you do that. Crab fixes this by going over every function and changing the name to include the parameter types and length of parameters. A function named `foo` that takes an int turns into `fooint1` in the machine code. 

### Codegen
The most fun part of a compiler is codegen. After all this work, you can take the final AST with all the conversions, and turn it into real code. LLVM is quite nice, and we can create a global module that we just add to as we go over the tree. For example, the code for generating an arithmetic operation goes as such:

```ocaml
 match op with 
            | "+"   -> a_gen_op build_add build_fadd e1 e2 "addtmp"
            | "-"   -> a_gen_op build_sub build_fsub e1 e2 "subtmp"
            | "*"   -> a_gen_op build_mul build_fmul e1 e2 "multmp"
            | "/"   -> a_gen_op build_sdiv build_fdiv e1 e2 "divtmp"

...

let a_gen_op func_int func_f e1 e2 name =
        let expr1 = codegen_expr ctx e1 in let expr2 = codegen_expr ctx e2 in
        (* We assume that the Type Checking 
            has assured that these are the same type *)
        let func = match e1.tp with
            | TInt      -> func_int
            | TFloat    -> func_f
            | _         -> assert false
        in 
        (func expr1 expr2 name builder) 
```

OCaml is pretty readable in my opinion, so I think the code is self-explanatory. LLVM allows us to use functions like `build_add` or `build_fdiv` using the `builder` that is defined globally. Maybe in another blog post, I will explain how LLVM does what it does because it is fascinating. But essentially, LLVM has some simple operations that we can use to build up our actual programs. Things like jumps and math operations are all we need to create loops and conditionals. 

# A "completed" project
One of the reasons I'm very proud of Crab, despite it being fairly rudimentary and having a couple of silly design mistakes, is because I finished it. There's a lot more that I think I can do with Crab, it's potential is unlimited and unexplored, but it is useable. There are man pages, you can use `make` to install it to your computer, and there is even a python script that handles linking and adding the standard library (just some printing functions). It is very difficult to take a project that you started from complete scratch and get it to a stage where you can stop working on it and be proud of it. Crab isn't much, I love compilers and I've done some fun projects that are not complete that do some of the more complicated things, like type inference or optimization that Crab has not touched, but unlike those, Crab is complete and that is why it is one of my favorite projects that I've undertaken. 