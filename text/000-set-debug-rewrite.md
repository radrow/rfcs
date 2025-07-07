- Title: `Set Debug Rewrite` for rewrite strategies

- Drivers: Radosław Rowicki (@radrow) / radrowicki at gmail

----

# Summary

Introduce a way to debug generalized rewriting via `autorewrite` and
`rewrite_strat`. We already have this for `auto` and friends (aka `debug auto`).

# Motivation

This addresses the feature request from
[rocq/9285](https://github.com/rocq-prover/rocq/issues/9285).

The motivation are my tears shed during debugging rewrite scripts. Currently,
there is completely no way to inspect what is happening inside rewrite scripts,
such as what lemmas are tried, which fail and why. The only feedback you get is
the final result which is either the ((wrongfully)) updated term, or an
information that the tactic "failed".

Moreover, debugging performance is just hopeless.

# Detailed design

I imagine the output to look about like:

```coq
Parameter T : Set.
Parameters A B C : T.

Parameter rab : A = B.
Parameter rbc : forall x, x = B -> x = C.

Hint Rewrite -> rab rbc using auto : rew.

Goal (C = A) <-> True.
  rewrite_strat (innermost hints rew).

(*
enter: C = A <-> True.
1. enter: C = A
1.1. skip: eq
1.2. skip: T
1.3. enter: C
1.3. fail (hint rew): rab (* no match *)
1.3. fail (hint rew): rbc (* assumption *)
1.4. enter: A
1.4. success (hint rew): rab
1.4. progress: B
1. progress: C = B
progress: C = B <-> True
*)
```

The log tree is to be read:

- Depth (horizontal) follows the traversal over the expression with numbers
  indicating subterm's index.
- Span (vertical) follows execution of the rewrite script.

Entries:

- `enter` shows the term under rewrite. Printed every time the script focuses on
  a (sub)term.
- `skip` works like `enter`, but indicates a boring failure or no progress.
- `fail` and `success` report result of a primitive strategy:
  - `(term ->): x` for explicit term `x`, ltr (vv with `<-`)
  - `(hint db): x` for term `x` coming from hint database `db`
  - `(fold): x` ... (you get the point)

  Additionally, `fail` displays the reason for failure:
  - `no match` self explanatory
  - `assumption` when a premise of a rewrite lemma could not be solved
- `progress` shows the updated (sub)term when progress is made

The `assumption` log could be upgraded to show what the failed premise looked
like. This would be helpful if previously successful assumptions instantiated
some evars.

The `hint` log might get stupidly big. Maybe it would be wise to drop the `no
match` log in this case? It could also be tweaked via some option whether we
want to see any attempts except the success, ideally per-hintdb.

As for `skip`, I like the criterion of skipping no-progress/fails when there was
no unification call nor evar instantiation. That can also be tweakable to some
extent — sometimes you want to see why something was skipped. For printing, we
may not want to print skipped subterms, but just batch their log entries:
`1.{1,2}. skip`. Or maybe just not print it at all.

I'm not sure yet how to nicely visualise evar unification. This can be a second
iteration though.

Bonus: a `dbg` rewstrategy that prints the term under focus with some optional
message (though it's a next-iteration feature imo).

#### Performance / profiling

Regarding debugging performance, I'd consider it a separate task. I see it as a
`time` rewstrategy that at the end produces stats:

- Prettyprint of the timed strategy and its in-code location (to distinguish
  different profiles of same-looking strats)
- The number of times it was called
- The cumulative time spent inside
- The cumulative time spent solving premises of rewrite lemmas
- (optionally/tweakable) detailed profile for each individual call:
  - The term under focus
  - What was the result (just progress/failure/nothing)
  - How much time it took overall
  - How much time was spent solving premises of rewrite lemmas

#### Implementation

In `rewrite.ml` the `strategy` type carries state. When debug is enabled, it
could just build up a log.

# Drawbacks

Overhead? But I don't think it'll be drastic.

# Alternatives

I will likely draw inspiration from `debug auto`.

# Unresolved questions

I am open to suggestions on the output format and contained info.


