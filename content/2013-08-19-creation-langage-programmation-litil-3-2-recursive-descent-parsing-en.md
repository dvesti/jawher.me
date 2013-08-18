date: 2013-08-19
slug: creation-langage-programmation-litil-3-2-recursive-descent-parsing
title: The Litil chronicle - Chapter 3.2, Recursive Descent Parsing
lang: en

In [The previous post](http://jawher.me/2013/08/11/creation-langage-programmation-litil-3-1-introduction-parsing/), I talked about some of the theory behind parsing.
This post however will be about how to actually write down a parser, which given a list of tokens generated by the lexer would generate an AST.

For example, given the following Litil program:

```litil
let bool2int x = if x then 1 else 0
```

Feeding it to the lexer would generate the following token stream:

1. `KEYWORD(let)`
2. `NAME(bool2int)`
3. `NAME(x)`
5. `SYM(=)`
6. `KEYWORD(if)`
6. `NAME(x)`
7. `KEYWORD(then)`
8. `NUM(1)`
7. `KEYWORD(else)`
8. `NUM(0)`

Feeding this token stream to the parser we're going to create would generate this AST[^1]:

{% dot litil-parser-ast0.png
graph GG {
    rankdir=LR  
    graph [rankstep=0]
    node [shape=record, fontname=monospace]
    
    EName1 [label="<o> EName|x"]
    ENum2 [label="<o> ENum|1"]
    ENum3 [label="<o> ENum|0"]
    EIf4 [label="<o> EIf|<cond> condition|<then> then|<else> else"] EIf4:cond -- EName1:o
    EIf4:then -- ENum2:o
    EIf4:else -- ENum3:o
    Let5 [label="LetBinding|name: bool2int|args: x|<is> instructions"]
    Let5:is -- EIf4:o
}
%}



As we've seen in the previous post, there are different kinds of parsers: Top-down (LL for example), Bottom-up (LR for example) and many concrete methods and algorithms to implement them.

I chose to use the [recursive descent parsing](http://en.wikipedia.org/wiki/Recursive-descent_parser) in litil for the following reasons:

* Such parsers are easy to write by hand
* The resulting code is readable and easy to evolve and maintain

## From grammar to parser

What follows is how to create a recursive descent parser from an EBNF grammar.

We're going to create a parser for a subset of litil.
Here's its EBNF grammar:

```
program → letBinding*

letBinding → 'let' NAME NAME* '=' expression
expression → 'if' expression 'then' expression 'else' expression | NAME | 'true' | 'false' | NUM
```

In the language defined by the grammar above, a program is a set of let bindings.  
A let binding starts with the `let` keyword followed by a name, an optional list of arguments, the `=` symbol followed by an expression.  

An expression can either be a name (referencing a variable by its name), a numeric literal (5, 42, etc.), a boolean literal (`true` or `false`) or an `if` expression.  
An `if` expression starts with the `if` keyword followed by a condition (which is an expression).  
The branch values are specified as an expression after the `then` and `else` keywords.

Some examples of programs written in this language:

```litil
let x = 5

let b = true

let y = if true then 45 else x
```

Granted, it is not a very useful language (it can't even add 2 numbers together or compare them).  
However, its grammar is compact and simple and it will serve for the purpose of demonstrating how to write a recursive descent parser.

### The framework

We'll start by creating a class for the parser that takes a lexer as its constructor argument:

```java
public class LitilParser {

    private LookaheadLexer lexer;

    public LitilParser(LookaheadLexer lexer) {
        this.lexer = lexer;
    }
}
```

Next, we'll add a field that will hold the current token the lexer just read and a couple of functions that operate on it:

```java
public class LitilParser {

    private LookaheadLexer lexer;
    private Token token;

    public LitilParser(LookaheadLexer lexer) {
        this.lexer = lexer;
    }

    private boolean found(Token.Type key) {
        Token tk = lexer.peek(1);
        if (tk.type == key) {
            lexer.pop();
            token = tk;
            return true;
        } else {
            return false;
        }
    }

    private boolean found(Token.Type key, String value) {
        Token tk = lexer.peek(1);
        if (tk.type == key && tk.text.equals(value)) {
            lexer.pop();
            token = tk;
            return true;
        } else {
            return false;
        }
    }
}
```

The `found` function checks if the upcoming token's type from the lexer matches its argument.  
If that's the case, the token is consumed and stored in the `token` field, and `true` is returned.  
Else, no token gets consumed and `false` is returned.

Here's an example of how this function can be used:

```java
if(found(Token.Type.NAME)) {
    String name = token.value;
}
```

There's a second variant that also checks the upcoming token's value besides its type:

```java
if(found(Token.Type.KEYWORD, "let")) {
    // try and parse a let binding
}
```

We'll also need `expect`, which behaves like `found`, except that it throws an exception instead of returning false:

```java
public class LitilParser {

    private void expect(Token.Type key) {
        if (!found(key)) {
            throw error("Was expecting a token of type '" + key + "' but got " + lexer.peek(1));
        }
    }

    private void expect(Token.Type key, String value) {
        if (!found(key, value)) {
            throw error("Was expecting '" + value + "'(" + key + ")");
        }
    }

    private ParseException error(String msg) {
        Token where = token == null ? lexer.peek(1) : token;
        throw new ParseException(msg, lexer.getCurrentLine(), where.row, where.col);
    }
}
```


The `error` function constructs a `ParseException`, which holds an error message, the current line being parsed and the parse coordinates (row and column):

```java
public class LitilParser {

    private ParseException error(String msg) {
        Token where = token == null ? lexer.peek(1) : token;
        throw new ParseException(msg, lexer.getCurrentLine(), where.row, where.col);
    }
}
```

Here's the `ParseException` declaration:

```java
public class ParseException extends RuntimeException {

    public ParseException(String message, String line, int row, int col) {
        super(String.format("%s (@%d:%d)\n%s\n%s^\n", message, row, col, line, ParseException.indent(col - 1)));
    }

    private static String indent(int n) {
        StringBuilder res = new StringBuilder();
        for (int i = 0; i < n; i++) {
            res.append(" ");
        }
        return res.toString();
    }
}
```

`ParseException` is a runtime exception (unchecked) that formats its arguments into a handy display format.  
For instance, given this invalid input (due to a missing let binding name):

```litil
let = true
```

The `ParseException` would generate a message resembling the following:

```
ParseException: Was expecting a token of type 'NAME' but got SYM ('=') @ 2:5 (@2:5)
let = true
    ^
```

### Coding the production rules

Once we have the basic framework I showed above, converting the EBNF grammar into a parser becomes as easy as:

1. Create a method for every production rule using the left hand side non terminal's name as the method's name
2. For the right hand part:
    1. A terminal maps to a call to `expect`
    2. A non-terminal maps to a call to its production rule method
    3. A choice (`|`) or an optional part (`?`) is handled with the `found` primitive and the host language's `if` construct
    4. A repetition is handled with the host language's `while` construct

Let's apply this to the grammar we defined above:

```
program → letBinding*
letBinding → 'let' NAME NAME* '=' expression
expression → 'if' expression 'then' expression 'else' expression | NAME | 'true' | 'false' | NUM
```

Applying the rule 1, we create 3 methods, one for every product rule:

```java
public void program() {
    // TODO
}


public void letBinding() {
    // TODO
}

public void expr() {
    // TODO
}
```

Let's implement the `program` method.  
As a reminder, here's the right hand part of its production rule:

```
program → letBinding*
```

i.e. zero or more repetitions of the letBinding non terminal.  
Applying the rule 2.4, we know we'll use a `while` loop the handle the repetition.
We'll also use a special feature in litil's lexer, which produces a `NEWLINE` token  for every non-empty line in the program.  
Here's how `program` should be implemented:

```java
public void program() {
    while (found(Token.Type.NEWLINE)) {
        letBinding();
    }
}
```

Or in english:

* as long a we get a `NEWLINE` token: that's the `while(found(NEWLINE))` part
* parse a let binding: this is done by simply calling the `letBinding` method (rule 2.2)

How about `letBinding` ?

Here's its production rule:

```
letBinding → 'let' NAME NAME* '=' expression
```

This cleanly maps to the following java code:

```java
public void letBinding() {
    expect(Token.Type.KEYWORD, "let");
    expect(Token.Type.NAME);
    while (found(Token.Type.NAME)) {
        // NOP
    }
    expect(Token.Type.SYM, "=");
    expr();
}
```

We start by handling the 'let' terminal.
We simply apply the rule 2.1 by calling `expect` to *expect* the `let` keyword.  
Same for the name that follows it. We simply call `expect(NAME)`.

We then have to handle `NAME*` (the optional arguments list). This is achieved with a `while` loop and the `found` primitive (rule 2.4).  
We then *expect* the `=` symbol and finally, parse an expression by calling the `expr` method.

Here's the production rule for an expression:

```
expression → 'if' expression 'then' expression 'else' expression | NAME | 'true' | 'false' | NUM
```

To handle the choice, we apply rule 2.3 and use the `if` construct with the `found` primitive:

```java
private void expr() {
    if (found(Token.Type.KEYWORD, "if")) {
        expr();
        expect(Token.Type.KEYWORD, "then");
        expr();
        expect(Token.Type.KEYWORD, "else");
        expr();
    } else if (found(Token.Type.NAME)) {
        //NOP
    } else if (found(Token.Type.NUM)) {
        //NOP
    } else if (found(Token.Type.BOOL)) {
        //NOP
    } else {
        throw error("Unrecognized expression");
    }
}
```

That's it.
Seriously, that's it.
We just coded a recursive descent parser for the grammar given above.

To actually run this parser, we can use this snippet:

```java
public static void main(String[] args) throws Exception {
    String code = "let y = if true then 45 else x"
    Reader reader = new StringReader(code);
    LitilParser p = new LitilParser(new LookaheadLexerWrapper(new StructuredLexer(new BaseLexer(reader), "  ")));
    p.program();
}
```

Simply instantiate a lexer and a parser, and call the `program` method to start the parsing process.

### AST

There still one thing missing though:
in its current form, the parser can only ensure that a program is valid according to a grammar or throw a `ParseException` if it isn't.  

That's useful in itself, but what we'd really like is to have it produce an AST so we can do more useful stuff later, like type checking or executing it for example.

In this case, and according the EBNF grammar defined above, we'll define these AST node types:

#### Program
A program is a list of let bindings:

```java
public class Program {
    public final List<LetBinding> instructions;

    public Program(List<LetBinding> instructions) {
        this.instructions = instructions;
    }
}
```

#### LetBinding
A let binding can be described with an object containing the following properties:

| Property | Type           | Description                                            |
| -        | -              | -                                                      |
| name     | `String`       | the name of the the variable or function being defined |
| args     | `List<String>` | the list of argument names                             |
| body     | `Expression`   | the expression forming the value of the let binding    |

or in Java:

```java
public class LetBinding {
    public final String name;
    public final List<String> args;
    public final Expression body;

    public LetBinding(String name, List<String> args, Expression body) {
        this.name = name;
        this.args = args;
        this.body = body;
    }
}
```

#### Expression
Now, expressions are a bit trickier because of the choice in the production rule:

```
expression → 'if' expression 'then' expression 'else' expression | NAME | 'true' | 'false' | NUM
```

So we're going to need a base type:

```java
public abstract class Expression {
}
```

And a sub class for every choice:

##### EIf

`EIf` should contain the following properties:

| Property | Type         | Description                                             |
| -        | -            | -                                                       |
| cond     | `Expression` | the if condition                                        |
| thenExpr | `Expression` | the expression to return when the `then` branch matches |
| elseExpr | `Expression` | the expression to return when the `else` branch matches |

Which translates to the following Java code:

```java
public class EIf extends Expression {
    public final Expression cond;
    public final Expression thenExpr;
    public final Expression elseExpr;

    public EIf(Expression cond, Expression thenExpr, Expression elseExpr) {
        this.cond = cond;
        this.thenExpr = thenExpr;
        this.elseExpr = elseExpr;
    }
}
```

##### EName
Just a class with a name field:


```java
public class EName extends Expression {
    public final String name;

    public EName(String name) {
        this.name = name;
    }
}
```

##### EBool
Same for boolean literals expressions:

```java
public class EBool extends Expression {
    public final boolean value;

    public EBool(boolean value) {
        this.value = value;
    }
}
```

##### ENum
And also the same for numeric literals expressions:

```java
public class ENum extends Expression {
    public final long value;

    public ENum(long value) {
        this.value = value;
    }
}
```

### Generating the AST

Now that we have defined our AST node types, we need to make the parser construct it.

We'll start by modifying the parse methods to return the correct AST type instead of `void`.

For example, where `expr()` used to return void:


```java
private Expression expr() {
    :
    :
}
```

We'll make it return `Expression` instead:

```java
private Expression expr() {
    :
    :
}
```

We'll also need to modify its implementation to actually return an `Expression`.

Let's start with the `if` expressions parsing.  
Here's what we have got so far:



```java
if (found(Token.Type.KEYWORD, "if")) {
    expr(); // parse the if condition
    expect(Token.Type.KEYWORD, "then");
    expr(); // parse the then result
    expect(Token.Type.KEYWORD, "else");
    expr(); // parse the else result
} else 
```

What we'll need to do is construct an `EIf` node using the condition and branches expressions:

```java
if (found(Token.Type.KEYWORD, "if")) {
    Expression cond = expr();
    expect(Token.Type.KEYWORD, "then");
    Expression thenExpr = expr();
    expect(Token.Type.KEYWORD, "else");
    Expression elseExpr = expr();
    return new EIf(cond, thenExpr, elseExpr);
} else 
```

That was the hardest one.  
The remaining ones (names, numeric and boolean literals) can be implemented as follow:

```java
 else if (found(Token.Type.NAME)) {
    return new EName(token.text);
} else if (found(Token.Type.NUM)) {
    return new ENum(Long.parseLong(token.text));
} else if (found(Token.Type.BOOL)) {
    return new EBool(Boolean.parseBoolean(token.text));
} else 
```

Now, here's how the complete `expr` method:

```java
private Expression expr() {
    if (found(Token.Type.KEYWORD, "if")) {
        Expression cond = expr();
        expect(Token.Type.KEYWORD, "then");
        Expression thenExpr = expr();
        expect(Token.Type.KEYWORD, "else");
        Expression elseExpr = expr();
        return new EIf(cond, thenExpr, elseExpr);
    } else if (found(Token.Type.NAME)) {
        return new EName(token.text);
    } else if (found(Token.Type.NUM)) {
        return new ENum(Long.parseLong(token.text));
    } else if (found(Token.Type.BOOL)) {
        return new EBool(Boolean.parseBoolean(token.text));
    } else {
        throw error("Unrecognized expression");
    }
}
```

How about the `let` binding construct ?
As a reminder, here's how we coded the `let` parsing mthod:

```java
public void letBinding() {
    expect(Token.Type.KEYWORD, "let");
    expect(Token.Type.NAME);
    while (found(Token.Type.NAME)) {
        // NOP
    }
    expect(Token.Type.SYM, "=");
    expr();
}
```

To get it to produce an AST node, we start by changing its return type from `void` to `LetBinding`:

```java
public LetBinding letBinding() {
    
}
```

And then, same as with the `if` parsing method, we're going to collect the necessary information to construct the result:

* the `let` binding name
* the arguments names
* the body's expression

The `let` binding name was parsed using the `expect` method:

```java
expect(Token.Type.KEYWORD, "let");
expect(Token.Type.NAME);
```

The `expect` function, as explained earlier, reads the next token produced by the lexer, and checks that its type and value match its arguments.
If a match is found, the consumed token is then stored in the `token` field.

Hence, the `let` binding's name can be retrieved after the `expect` call in `token.text`:

```java
expect(Token.Type.KEYWORD, "let");
expect(Token.Type.NAME);
String name = token.text;
```

The same applies to the `found` method, i.e. if it matches, the cosumed token is stored in the `token` field.  
Thus, to collect the arguments names, we need to modify the empty `while` loop's body to add the current token's text into a list:

```java
List<String> args = new ArrayList<String>();
while (found(Token.Type.NAME)) {
    args.add(token.text);
}
```

It only remains to store the `let` bindings's body expression into a variable:

```java
Expression body = expr();
```

And finally to construct and return a `LetBinding` instance using the collected name, argument list and body:

```java
return new LetBinding(name, args, body);
```

And we're done !
Here's the complete listing of the `let` parsing method:

```java
public LetBinding letBinding() {
    expect(Token.Type.KEYWORD, "let");
    expect(Token.Type.NAME);
    String name = token.text;

    List<String> args = new ArrayList<String>();
    while (found(Token.Type.NAME)) {
        args.add(token.text);
    }
    expect(Token.Type.SYM, "=");
    Expression body = expr();
    return new LetBinding(name, args, body);
}
```

Easy peasy !

I think you got how it works by now.
So I'll simply paste the modified version of the program parsing method below:

```java
public Program program() {
    List<LetBinding> instructions = new ArrayList<LetBinding>();
    while (found(Token.Type.NEWLINE)) {
        instructions.add(letBinding());
    }
    return new Program(instructions);
}
```

Now that the different parsing methods, and mainly `program` return the AST, we can go back to the `main` function to use the return value:

```java
public static void main(String[] args) throws Exception {
    String code = "let y = if true then 45 else x"
    Reader reader = new StringReader(code);
    LitilParser p = new LitilParser(new LookaheadLexerWrapper(new StructuredLexer(new BaseLexer(reader), "  ")));
    
    Program program = p.program();
    for(Instruction instr: program.instruction) {
        // Do amazing things with the instructions !
    }
}
```

And there you have it, a recursive descent parser that ensures the validaity of a program according to a grammar and produces an AST representing the structure of the program.

[^1]: As a side note, the AST tree images in this post were generated using a visitor that transforms a parsed litil AST into a dot file.