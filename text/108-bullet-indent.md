- Title: Indentation based bullet behavior

- Drivers: Pierre Roux (@proux01)

----

# Summary

We offer a new bullet behavior using indentation to circumvent
limitations of the current one, while helping to check indentation of
proof scripts.

# Motivation

Bullets (typically `-`, `+`, `*`) were initially introduced by the
MathComp library. They were at that time mere comments, without any
semantic check by Coq. Checks were later implemented in Coq but with a
semantic dramatically incompatible with the established usage in
MathComp. We try here to devise a new semantic that would enable to
compile MathComp with no or minimal changes to its code base.
Moreover, the new semantics should essentially be a conservative
extension of the current one, at least for already reasonnably
indented proof scripts.

A typical limitation of the current behavior is the following kind of
scripts, leading to ever increasing indentation:

```Coq
Goal True /\ True /\ True /\ True /\ True.
Proof.
split.
- exact I.
- split.
  + exact I.
  + split.
    * exact I.
    * split.
      -- exact I.
      -- exact I.
Qed.
```

Whereas one would rather like to write something like:

```Coq
Goal True /\ True /\ True /\ True /\ True.
Proof.
split.
  exact I.
split.
  exact I.
split.
  exact I.
split.
- exact I.
- exact I.
Qed.
```

This scheme happens a lot in actual mathematical proofs with a few
simple side conditions, handled first, then a -- more difficult --
main goal, constituting the bulk of the proof.

# Detailed design

We have a stack of

```ocaml
type indent =
  | IndentBullet of bullet_type * int
  | IndentTactic of int
```

When encountering new bullet `b` at indentation `indent`:
* look in stack for same bullet
  * if found, check for same indentation,
    if not same indentation, warn about indentation
  * if not found, if top of stack is `IndentTactic pre_indent`,
    check that `indent = pre_indent`, otherwise warn about indentation
* check that at least two goals are focused
* then pop stack, adding `IndentBullet (b, indent)` on top of stack
  and focus first goal, as of today

When encountering tactic at indentation `indent`:
* if top of stack is `IndentBullet (_, pre_indent)`,
  check that `indent >= pre_indent` (otherwise warn about indentation)
  and add `IndentTactic indent` of top of stack
* otherwise top of stack is `IndentTactic pre_indent`
  * if `indent = pre_indent`, nothing to do
  * if `indent > pre_indent` (let's call that "positive indentation"),
    check exactly two focused goals (otherwise error too many goals),
    focus first and add `IndentTactic indent` of top of stack
  * if `indent < pre_indent` (let's call that "negative indentation"),
    check for absence of focused goal (raise an error if any),
    unfocus and pop stack, warn about indentation if `indent` doesn't match top of stack,
    check for single focused goal and add `IndentTactic indent` on top of stack

Implem details:
* happens mostly in `proof_bullet.ml` + tracking of indentation in higher levels
  and call of `Proof_bullet.push` with a new `BulletIndent` type for each tactic
* add `Proofview.nb_goals` (look at `Proofview.finished`)
* add `Proof.nb_focused_goals` (look at `Proof.no_focused_goal`)

## Examples

### Warning and errors

```Coq
Set Default Goal Selector "!".
Set Bullet Behavior "Indent".

Goal True /\ True /\ (True /\ True).
Proof.
- (* Error: cannot use bullet on single goal
     (currently accepted) *)
Abort.

Goal True /\ True /\ (True /\ True).
Proof.
split.
-
  + (* Error: cannot use bullet on single goal
       (currently accepted) *)
Abort.

Goal True /\ True /\ True.
Proof.
split.
  - (* Warning: wrong indentation *)
Abort.

Goal True /\ True /\ True.
Proof.
  split.
- (* Warning: wrong indentation *)
Abort.

Goal True /\ True /\ True.
Proof.
split.
- exact I.
  - (* Warning: wrong indentation *)
Abort.

Goal True /\ True /\ True.
Proof.
  split.
  - exact I.
- (* Warning: wrong indentation *)
Abort.

Goal True /\ True /\ True.
Proof.
  split.
  -
exact I.
(* Warning: wrong indentation *)
Abort.

Goal True /\ True /\ True.
Proof.
  split; [|split].
  - exact I.
exact I.
(* Warning: wrong indentation *)
(* Error: 2 remaining goals, negative indentation can only be used on last goal *)
Abort.
```

### Working examples

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
  exact I.  (* positive indentation, exactly two goals, focusing first *)
split; [|split].
- exact I. (* regular use of bullets *)
- exact I.
exact I.  (* last subgoal, using negative indentation *)
Qed.

Goal True /\ True.
Proof.
split.
  exact I.  (* positive indentation, exactly two goals, focusing first *)
exact I.  (* last subgoal, using negative indentation *)
Qed.
```
