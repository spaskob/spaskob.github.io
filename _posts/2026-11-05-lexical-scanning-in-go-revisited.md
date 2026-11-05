---
layout: post
title: "Rob Pike’s Lexical Scanning in Go, Revisited"
date: 2026-11-05
render_with_liquid: false
---

Rob Pike's 2011 talk, [Lexical Scanning in Go](https://go.dev/talks/2011/lex.slide), is a very instructive example of the general theme of [Concurrency is not parallelism](https://www.youtube.com/watch?v=oV9rvDllKEg). It's one of my favorite talks on Go, but I find it a bit hard to follow. Its main idea, a lexer built from functions that return their successors, gets tangled up with a second idea: using a channel to coordinate the lexer and the parser.

This post separates the two. We'll build up to Pike's design starting from a plain lexing loop.

## A Tiny Template Language

Let's simplify Pike's original problem. Suppose we're building a template language that allows ordinary text to contain expressions enclosed in double curly braces:

```text
Hello, {{name}}!
Is {{x+42}} really the answer?
```

Inside the braces, we'll support identifiers, integer constants, and addition or subtraction.

Conceptually, processing a template involves two steps. First, a **lexer** scans the input and produces a list of tokens. Then a **parser** transforms those tokens into an *abstract syntax tree* (AST), representing the structure of the template.

For our tiny language, we can represent a token with a Go struct:

```go
type token struct {
    typ string // e.g. "IDENT", "NUMBER", "TEXT"
    val string // e.g. "x", "42", "Hello, "
}
```

Pike uses an integer enum for token types; we'll use strings to keep things simple. Following his terminology, we'll call the delimiters `{{` and `}}` *meta-strings*, with token types `LEFT_META` and `RIGHT_META`.

For example, given:

```text
The answer is {{x+42}}.
```

the lexer produces this tokenization:

```text
TEXT        "The answer is "
LEFT_META   "{{"
IDENT       "x"
PLUS        "+"
NUMBER      "42"
RIGHT_META  "}}"
TEXT        "."
EOF
```

The parser then turns this flat sequence of tokens into a tree:

```text
Template
├── Text("The answer is ")
├── Expression
│   └── Add
│       ├── Ident("x")
│       └── Number(42)
└── Text(".")
```

Notice the distinction. The lexer identifies individual pieces of the input, but knows nothing about their relationships. The parser recognizes that `x+42` is an addition expression, with `x` and `42` as its operands. Once we have this tree, evaluating the template is straightforward: substitute the value of `x`, compute the addition, and combine the result with the surrounding text.

For our purposes, though, we'll focus entirely on the lexer.

The simplest interface between lexer and parser would be to scan the entire input, collect all tokens into an array, and then hand that array to the parser. That works fine for a small template. But for a large document, we're allocating memory for every token before the parser can begin processing them. Ideally the parser could simply ask for the next token whenever it needed one.

```go
tok := lex.NextToken()
```

We'd like the lexer to produce one token, return control to the parser, and resume scanning when the parser asks for another.

## 1. The Straightforward Solution: A Loop

The lexer has two modes of operation. Outside the braces, it scans ordinary text. Inside, it scans expressions. We can represent that distinction with a boolean state.

```go
type lexer struct {
    input  string
    pos    int
    inExpr bool  // tracks if we are inside a {{...}} block.
}
```

And write `NextToken()` as one loop:

```go
func (l *lexer) NextToken() token {
    for !l.atEOF() {
        if !l.inExpr {
            // We're scanning ordinary text.
            if l.startsWith("{{") {
                l.advance(2)
                l.inExpr = true
                return token{"LEFT_META", "{{"}
            }

            start := l.pos
            for !l.atEOF() && !l.startsWith("{{") {
                l.next()
            }
            return token{"TEXT", l.input[start:l.pos]}
        }

        // We're scanning an expression.
        if l.startsWith("}}") {
            l.advance(2)
            l.inExpr = false
            return token{"RIGHT_META", "}}"}
        }

        if l.isSpace() {
            l.next()
            continue
        }

        return l.scanExprToken()
    }

    if l.inExpr {
        return token{"ERROR", "unterminated expression"}
    }
    return token{"EOF", ""}
}
```

Here, `l.pos` is our current position in the input, and `l.next()` advances it by one character. The helper `scanExprToken()` recognizes identifiers, numbers, and operators. We'll leave those uninteresting details out.

The function scans until it recognizes one token, returns it, and remembers where it stopped. The next call continues from the same position. For our little template language, this is probably good enough. The boolean `inExpr` keeps track of which part the lexer should execute next. Every time around the loop, we inspect that variable and dispatch to the appropriate scanning logic. It encodes a tiny state machine.

This is closely related to an observation of Russ Cox in [Storing Data in Control Flow](https://research.swtch.com/pcdata). Sometimes we introduce variables whose purpose is to record information that could instead be represented by the structure of the program itself.

This approach will progressively get harder to maintain and understand if we start growing the template language. Suppose we add quoted strings inside expressions:

```text
Message: {{"Hello, world"}}
```

And escaped characters:

```text
{{"He said \"hello\""}}
```

Perhaps we also introduce comments or more complicated expressions. Now we have several distinct scanning behaviors, each with its own rules and transitions. We can certainly keep extending our loop, adding more flags or introducing an explicit state enum. But `NextToken()` is gradually becoming a central dispatcher for an increasing collection of different activities.

And there's something slightly redundant about that arrangement. When we're scanning ordinary text, we already know we're in the text-scanning state. When we're scanning an expression, we already know we're inside an expression. Why must we repeatedly consult a variable to discover which code should execute? Could we express that distinction using the program's own control flow?

### Turning States Into Functions

Let's give each scanning state its own function. Rather than maintaining an explicit state variable, a function can simply call the function representing the next state.

Here is an implementation of the function for scanning ordinary text. Because it hands control directly to the next state function instead of returning to the parser, it passes recognized tokens to a helper method, `l.emit(...)`, which we'll define in a moment:

```go
func lexText(l *lexer) {
    start := l.pos

    // Scan until we find {{ or reach EOF.
    for !l.atEOF() && !strings.HasPrefix(l.input[l.pos:], "{{") {
        l.next()
    }

    // Emit any text we scanned.
    if l.pos > start {
        l.emit(token{typ: "TEXT", val: l.input[start:l.pos]})
    }

    if l.atEOF() {
        l.emit(token{typ: "EOF"})
        return
    }
    // We know we are at '{{' so we call the function that handles left meta directly.
    lexLeftMeta(l)
}
```

When the lexer encounters `{{`, it transitions to `lexLeftMeta`, a short state that consumes the opening delimiter and then hands control to `lexExpr`, the state responsible for scanning everything inside the expression:

```go
func lexLeftMeta(l *lexer) {
    l.pos += len("{{")
    l.emit(token{typ: "LEFT_META", val: "{{"})

    lexExpr(l)
}
```

And inside an expression:

```go
func lexExpr(l *lexer) {
    if l.consume("}}") {
        l.emit(token{typ: "RIGHT_META", val: "}}"})
        // We are out of '{{...}}' block so we are back to processing plain text.
        lexText(l)
        return
    }

    // Scan and emit one expression token.
    l.emit(l.scanExprToken())

    lexExpr(l)
}
```

Notice that the `inExpr` flag is gone. Instead of storing the mode in a struct field and branching on it in a loop, the lexer tracks its state through the control flow of the recursion itself. Each function represents a state: calling a different function transitions to a new state, and `lexExpr` calling itself scans the next token in the same state.

Because every transition is a call at the end of a function, the code maps directly onto a right-regular grammar for the language:

```text
Text     -> CHAR* EOF
          | CHAR* LeftMeta
LeftMeta -> "{{" Expr
Expr     -> "}}" Text
          | TOKEN Expr
```

But we've introduced a complication. What happened to our convenient `NextToken()` interface? In this version, once we call `lexText(l)`, the functions keep calling one another until the input is exhausted. There's no obvious point where we can return one token to the parser and later resume the computation from exactly where it stopped.

For now, we can take the easy way out and collect tokens in an array, just as we discussed earlier. Assume the lexer has a `tokens []token` field, and `emit` simply appends to it:

```go
func (l *lexer) emit(t token) {
    l.tokens = append(l.tokens, t)
}
```

But that means we're back to scanning the entire input before parsing it, allocating space for every token along the way. We've gained a much cleaner decomposition into states, but we've lost the incremental behavior we started with.

And there's another problem.

## 2. The Trouble with Tail Recursion

Look at the final line of `lexExpr`:

```go
lexExpr(l)
```

This is a *tail call*: the function has nothing left to do after calling itself. In a language with proper tail-call optimization (TCO), the compiler could replace that call with a jump, reusing the existing stack frame.

Unfortunately, Go doesn't guarantee this optimization. This is partly a deliberate design choice: eliminating tail calls would also eliminate their stack frames from stack traces, making debugging harder. As [Russ Cox explained in 2011](https://groups.google.com/g/golang-dev/c/8weyKlQIZt0), the Go team valued useful stack traces. Features such as `defer`, which may require additional work when a function returns, also complicate tail-call elimination.

Without tail-call elimination, each recursive transition ordinarily adds another frame, and stack consumption grows with recursion depth rather than remaining constant. A sufficiently long input can exhaust the stack and crash the program.

One solution is to replace the recursion with an explicit loop and a switch statement tracking the current state. This is precisely the conventional approach Pike introduces in [slides 16 and 17](https://go.dev/talks/2011/lex.slide#16). Adapted to our little lexer, it might look like this:

```go
for state != done {
    switch state {
    case text:
        state = scanText(l)
    case leftMeta:
        state = scanLeftMeta(l)
    case expr:
        state = scanExpr(l)
    }
}
```

Each action performs some work and returns an identifier representing the next state. The loop then examines that identifier to decide which action to execute next. This approach could also support an incremental `NextToken()`. We could store the current state in the lexer and run the loop only until a token becomes available. But it brings us back to a centralized dispatcher that needs to know about every state.

### A Brief Detour: Tail Recursion and State Machines

There's an interesting connection to compiler theory here. The `switch` statement isn't merely an arbitrary way to implement a state machine. It's closely related to how *mutual tail recursion* is eliminated.

Ordinary tail recursion is straightforward to eliminate. Consider:

```go
func countdown(n int) {
    if n == 0 {
        return
    }
    countdown(n - 1)
}
```

Because the recursive call is the last operation, we can replace it with a loop:

```go
func countdown(n int) {
    for n != 0 {
        n--
    }
}
```

Now consider two functions that call each other recursively:

```go
func even(n int) bool {
    if n == 0 {
        return true
    }
    return odd(n - 1)
}

func odd(n int) bool {
    if n == 0 {
        return false
    }
    return even(n - 1)
}
```

Both calls are in tail position. But now we have *mutual recursion*, involving two different functions. We can eliminate the recursion by introducing an explicit state:

```go
func isEven(n int) bool {
    state := "even"

    for {
        switch state {
        case "even":
            if n == 0 {
                return true
            }
            n--
            state = "odd"

        case "odd":
            if n == 0 {
                return false
            }
            n--
            state = "even"
        }
    }
}
```

Notice what happened. Each function became a case in the switch. Each recursive call became a state transition. And the arguments became variables maintained by the loop.

**This is structurally the same transformation as our lexer example.**

Of course, a compiler supporting proper tail calls wouldn't necessarily need the switch. It could instead generate machine code that jumps directly between the functions:

```text
even:
    if n == 0: return true
    n = n - 1
    jump odd

odd:
    if n == 0: return false
    n = n - 1
    jump even
```

So the `switch` isn't a clumsy workaround. It's a perfectly reasonable source-level implementation of tail-call elimination, particularly in a language like Go that doesn't guarantee the optimization. The question is whether we can achieve the same result without sacrificing the clarity of our original functions.

### Back to Pike's Criticism

The switch statement solves the stack problem. But it feels a bit awkward.

When an action finishes, **it already knows exactly which state should execute next**. Yet instead of transferring control directly to that state, we encode the decision as an identifier, return to the central loop, and use a switch statement to recover the corresponding action. In other words, we already know where we want to go, but we insist on consulting a dispatch table to get there. Pike describes this approach as *boring, repetitive, and error-prone*. Why do we need the switch at all?


## 3. Return the Next Function Instead of Calling It

If an action already knows which function should run next, why return an enum? Why not just return the next function itself?

In Go, functions are first-class values. A state doesn't have to execute its successor directly, nor does it need to encode its decision as an integer identifier. It can simply hand the successor function back to the caller: *"I'm done; here is who should take over next."*

Because each state function returns another state function, we can define the type recursively:

```go
type stateFn func(*lexer) stateFn
```

A state is a function that receives a lexer and returns another state function of the same type.

Now let's rewrite our previous example:

```go
func lexText(l *lexer) stateFn {
    // Scan and emit ordinary text.

    if l.atEOF() {
        l.emit(token{typ: "EOF"})
        return nil
    }

    return lexLeftMeta
}
```

The next state is still `lexLeftMeta`, but we return it rather than invoke it.

Similarly:

```go
func lexLeftMeta(l *lexer) stateFn {
    l.pos += len("{{")
    l.emit(token{typ: "LEFT_META", val: "{{"})

    return lexExpr
}
```

And:

```go
func lexExpr(l *lexer) stateFn {
    if l.consume("}}") {
        l.emit(token{typ: "RIGHT_META", val: "}}"})
        return lexText
    }

    l.emit(l.scanExprToken())

    return lexExpr
}
```

Compare the original recursive call:

```go
lexExpr(l)
```

with:

```go
return lexExpr
```

The state no longer executes its successor. It merely tells its caller what should execute next.

### The Trampoline Runner

Now we need something to invoke those returned functions:

```go
func run(l *lexer) {
    for state := lexText; state != nil; {
        state = state(l)
    }
}
```

That's the entire execution engine. It invokes the current state, receives its successor, and repeats until the returned state is `nil`.

This technique has a name: **[a trampoline](https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/)**. It's a standard pattern in programming-language theory for implementing tail-call elimination in languages that don't support it natively.

The name comes from the control flow: instead of diving deeper into a growing call stack, each function immediately returns up to the loop, which "bounces" execution into the next function. Because every state function returns before its successor is invoked, its stack frame is popped. The call stack remains constant, no matter how long the input is.

It neatly resolves both drawbacks we ran into earlier:
- **No unbounded recursion:** We eliminate the stack overflow risk of the mutual-recursion approach without requiring compiler-level tail-call optimization.
- **No central dispatch table:** Unlike the switch loop, there is no state enum or centralized dispatcher. The transition logic stays encapsulated inside the state that makes the choice.

## 4. Controlling Execution

Our trampoline runner executes until the input is exhausted. But we started with a different requirement: we want the parser to ask for tokens incrementally, one by one:

```go
tok := lex.NextToken()
```

How do we pause execution between tokens?

### Pike's 2011 Solution: A Goroutine and a Channel

Pike solved this coordination problem by introducing concurrency. Rather than running the lexer synchronously in the parser's thread, he ran the trampoline loop in its own goroutine and streamed tokens through a channel:

```go
type lexer struct {
    input string
    pos   int
    items chan token
}

func (l *lexer) emit(t token) {
    l.items <- t
}

func (l *lexer) run() {
    for state := lexText; state != nil; {
        state = state(l)
    }
    close(l.items)
}

func lex(input string) (*lexer, <-chan token) {
    l := &lexer{
        input: input,
        items: make(chan token),
    }
    go l.run()
    return l, l.items
}
```

The parser then consumes tokens directly from the channel:

```go
func (l *lexer) NextToken() (token, bool) {
    tok, ok := <-l.items
    return tok, ok
}
```

Because `l.items` is an unbuffered channel, every call to `emit` blocks the lexer until the parser calls `NextToken()`. Conversely, if the parser requests a token before one is ready, the parser blocks until the lexer reaches an `emit` call.

In his talk, Pike praised this as a triumph of concurrent design: the lexer and parser are written as independent sequential processes that coordinate through communication.

Yet from an engineering standpoint, this solution has distinct drawbacks:
- **Synchronization overhead:** Every token incurs the cost of a channel send and receive, memory synchronization, and potentially a goroutine context switch.
- **Resource leaks on early exit:** If the parser hits a syntax error and aborts parsing, it stops draining `l.items`. The background lexer goroutine remains permanently blocked on `l.items <- t`, leaking its stack and state. Preventing this leak requires additional machinery—such as a cancellation channel or context.
- **Unnecessary concurrency:** The lexer and parser do not actually need to run simultaneously. They operate in strict ping-pong lockstep. Using an independent thread of execution and synchronization primitives just to achieve pause-and-resume control flow is a heavy hammer.

### A Simpler Alternative: Synchronous Suspension

We don't need a goroutine to pause execution. With our trampoline, execution is already broken into discrete state transitions:

```go
state = state(l)
```

Instead of running that loop to completion inside a background goroutine, we can let the parser drive the loop directly. We store the current state function on the lexer struct:

```go
type lexer struct {
    input   string
    pos     int
    state   stateFn
    pending []token
}
```

Initialize `state` to `lexText`. For this version, let `emit` append to `pending` rather than sending to a channel.

Now `NextToken()` runs the state machine only until a token is produced:

```go
func (l *lexer) NextToken() (token, bool) {
    // Run the state machine until it emits a new token or finishes.
    for len(l.pending) == 0 && l.state != nil {
        l.state = l.state(l)
    }

    if len(l.pending) == 0 {
        return token{}, false
    }

    // Return the next available token.
    tok := l.pending[0]
    l.pending = l.pending[1:]

    return tok, true
}
```

In a full lexer, a single transition might emit multiple tokens (for example, closing an expression and immediately opening another). Using a slice for `pending` accommodates that flexibility. In our simplified example, each state emits at most one token per invocation, so the buffer never holds more than one item.

And unlike collecting every token in an array up front, memory usage remains minimal: we only retain the single pending buffer, the lexer position, and the current state pointer. If the parser encounters an error and stops calling `NextToken()`, the lexer simply gets garbage collected—no background goroutines to drain, no channels to close, and no leak risks.

However, this approach does establish a particular relationship between the lexer and parser: the parser drives execution by requesting tokens. That's perfectly reasonable for our purposes. The question of what happens when both computations want to retain control of their own execution is an interesting one—but we'll save that for another time.

## 5. Control Flow as Data: Continuations and Coroutines

Pike originally presented this lexer as an illustration of concurrency design: using a goroutine and a channel to decouple the lexer from the parser, demonstrating how concurrent structure can clarify a program even when the implementation runs sequentially.

Yet the core mechanism that makes the design work has little to do with concurrency. It rests on three standard concepts from programming-language theory:

### Continuations as First-Class Values
In standard execution, "what happens next" is implicit, maintained by the runtime on the call stack. When a function calls another, control transfers down the stack.

Returning the successor function turns that control transfer into a first-class value. Instead of executing the next step, the state function yields it as data: `return lexExpr`. The runner loop acts as a minimal interpreter for these continuations, repeatedly evaluating each function until the computation reaches a terminal state (`nil`).

### Trampolining as Tail-Call Elimination
Because Go does not guarantee tail-call optimization, direct mutual recursion between states grows the stack linearly with input size.

The trampoline (`for state != nil { state = state(l) }`) is the standard technique for eliminating tail calls in runtimes without native TCO. Because each state returns to the trampoline before its successor is invoked, its stack frame is popped immediately. The entire scan executes within a single, constant-depth stack frame, without requiring a central `switch` statement or state enumeration.

### Cooperative Suspension Without Language Coroutines
A common reason programmers reach for goroutines or language-level generators (`yield`) is to achieve cooperative suspension: allowing a producer to pause when an item is ready and resume when the consumer requests the next.

Pike's design achieves that same incremental execution synchronously. Because the trampoline introduces an explicit boundary between every state transition (`l.state = l.state(l)`), `NextToken()` can pause execution whenever `len(l.pending) > 0`, store `l.state` on the lexer struct, and return control to the parser. It provides the ergonomics of a generator without language-level coroutine support, thread synchronization, or channel allocations.

## Epilogue

There's one question we haven't really answered: which implementation is actually faster?

The original channel-based lexer has an obvious advantage on a multicore machine. While the parser is busy constructing the syntax tree, the lexer could already be scanning the next tokens. With two goroutines running on separate cores, lexing and parsing can overlap.

But there's a counterargument. Every token sent through a channel must pass through a thread-safe communication mechanism. The channel copies the token descriptor, performs synchronization, and may need to block and wake goroutines. For our small tokens, these costs could easily outweigh the benefit of parallelism. The synchronous `NextToken()` approach avoids most of that machinery.

So which wins? In September 2022, eleven years after his original talk, **Rob Pike himself removed the goroutines and channels from Go's standard-library template lexer**.

In the [commit message](https://go.googlesource.com/go/+/36760ca9fd3eaf2c79efef48e533b66da6c542bd), Pike wrote:

> The concurrent model for delivering tokens was fine for pedagogy

But the implementation had become increasingly troublesome as the package evolved. In [the original issue](https://github.com/golang/go/issues/53261), he explained that concurrency complicated interactions with other parts of the parser, could introduce races, and made the implementation harder to maintain.

His replacement was almost exactly the synchronous approach we've just explored: run the state machine until it produces a token, return that token to the parser, and resume the lexer when another token is requested.

In the benchmarks included with the change, execution time for the large-template parsing benchmark dropped from **12.22 ms to 6.77 ms — a 44.6% reduction**, while two much smaller benchmarks showed virtually no change.

That's not proof that synchronous lexers always outperform concurrent ones. A different parser, workload, buffering strategy, or amount of computation per token could produce a different result. But it's a striking outcome for the particular implementation Pike originally used to illustrate concurrency. There's also an independent investigation by [Eli Bendersky, published in June 2022](https://eli.thegreenplace.net/2022/performance-of-coroutine-style-lexers-in-go/), which found substantial overhead from per-token channel communication and experimented with buffered channels to reduce it.

Just as notable is what Pike *didn't* remove. The `stateFn` type, the functions returning their successors, and the tiny interpreter all survived.

---

**Further reading**

- [Rob Pike — Lexical Scanning in Go (2011)](https://go.dev/talks/2011/lex.slide)
- [Russ Cox — Storing Data in Control Flow (2023)](https://research.swtch.com/pcdata)
- [Florian Weimer — An Observation About Tail Calls (2011)](https://www.enyo.de/fw/notes/tail-calls.html)
- [Eli Bendersky — On Recursion, Continuations and Trampolines (2017)](https://eli.thegreenplace.net/2017/on-recursion-continuations-and-trampolines/)
- [Eli Bendersky — Performance of Coroutine-Style Lexers in Go (2022)](https://eli.thegreenplace.net/2022/performance-of-coroutine-style-lexers-in-go/)
