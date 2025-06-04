- Title: Assert outputs of vernacular commands and tactics within Rocq code

- Drivers: Radosław Jan Rowicki @radrow

----

# Summary

I would like to see and test outputs of vernacular commands without necessarily
going into interactive. More concretely, I propose to add a feature that lets me
paste the desired output of a particular command into Rocq code and make Rocq
verify whether it matches. The concept is similar to the existing `Fail`
command.

This is by the way a
[feature](https://florisvandoorn.com/carleson/docs/Lean/Elab/GuardMsgs.html) in
Lean (the docs are quite poor though).

# Motivation

### Testing

First, compiler and plugin testing. Right now
[rocq-prover/rocq](https://github.com/rocq-prover/rocq) has some awkward tests
that store reference outputs in separate files which are then compared to
produced outputs from `.v` samples. This feature would allow writing such tests
more clearly and without additional scripting.

Second, this aids testing Rocq code. Currently there exist workarounds for
particular cases, but there is no universal way to test things like `Check` on
modules, `Compute`, or `Ltac` scripts. A good example of this is the `Fail`
command which currently allows only checking that a command has failed, but not
the reason of the failure. Sure, the outputs may change occasionally and require
manual intervention, but that doesn't happen that often due to Rocq updates, and
if it's due to changes in code, then it's not a shock that you have to update
the test as well. Also fixing it is not really a big deal. Rocq and IDEs could
even support autoupdating reference outputs.

### Presentation

Having the ability to recall a definition makes Rocq scripts much more readable,
and guarantees resilience against updates. It also reassures the reader that the
output is not bogus — they just need to compile the project once, generate HTML
doc and read like a book (i.e. without an IDE).

### Education

Storing command outputs in code automatically audits tutorials, helping keeping
them up-to-date. Code editors could catch up and autogenerate such comments.
Inline outputs make it also easier to grasp a bigger picture of the env, which
is helpful while learning.

As for my experience, this feature was very handy while learning Lean.

# Detailed design

As discussed with @SkySkimmer in rocq-prover/rocq#20688, this could work
similarly to `Fail`. The tested command is preceded by `With Output str` where
`str` is a string literal containing the desired output.

```coq
Definition f x := x + 3.

With Output "
f = fun x : nat => x + 3
     : nat -> nat
"
Print x.
```

It should apply to tactics in proof mode as well:

```coq
Goal False.
  With Output "
   (* debug trivial: *)
    * assumption. (*fail*)
    * intro. (*fail*)
  "
  debug trivial.

  With Output "
    The command has indeed failed with message:
    The type has no constructors.
  "
  Fail constructor.
```

In the first iteration, the contents of the string literal must perfectly match
the command output modulo duplicated whitespaces.

### Possible improvements

Just ideas. To be discussed in more detail after this RFC is concluded.

1. Allow regexes.
2. Flag to turn this off — both in code and as a CLI parameter.
3. We can deduct how to parse output of some commands and allow more
   flexibility. For example, we could accept some minor term conversions in
   cases of `Check` and `Print`.

# Drawbacks

- Another restricted keyword (`With`).
- Might increase dependency on particular Rocq versions if misused. A flag to
  turn it off should solve some problems.

# Alternatives

### Attribute instead of command

We could use something like `#[outputs("blah")]` instead of a higher order command.

### Workarounds

- `Check` has a workaround as you can do

```coq
Check f : nat -> nat.
```

- `Print` can be hacked modulo conversions with

```coq
Check eq_refl f = fun x => x + 3.
```

but it does not let you test notations.

- For modules you can paste their type and assert they implement it. Meh.

- Sections I print one-by one. This works until I forget to print something.

- For other stuff, glhf.


# Unresolved questions

- Is `With Output` a good name?

