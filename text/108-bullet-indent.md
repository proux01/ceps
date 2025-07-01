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

We add a new bullet behavior:

```Coq
Set Bullet Behavior "Indent".
```

## Example

```Coq
Set Default Goal Selector "!".
Set Bullet Behavior "Indent".

Goal True /\ True /\ (True /\ True).
Proof.
split.
  exact I.  (* positive indentation, exactly two goals, focusing first *)
split; [|split].  (* last subgoal, using negative indentation *)
- exact I. (* regular use of bullets *)
- exact I.
exact I.  (* last subgoal, using negative indentation *)
Qed.
```

## Main ideas

In addition to current bullets:
* a *positive indentation* (a line being more indented than previous one)
  acts as a bullet when there are exactly two focused goals ;
* a *negative indentation* (a line being less indented than previous one)
  acts as a last bullet (when there is exactly one focused goal)

## Algorithm

We use a stack of

```ocaml
type indent =
  | IndentBullet of bullet_type * int
  | IndentTactic of int
```

When encountering a new bullet `b` at indentation `indent` we
essentially handle it as of today, with a few additional warnings:
* look in the stack for the same bullet `IdentBullet(b, pre_indent)`
  * when found, warn if `indent != pre_indent`
  * otherwise, if the top of the stack is `IndentTactic pre_indent`,
    warn if `indent != pre_indent`
* then, warn if a single goal is focused (the bullet is useless)
* then proceed with current algorithm for bullets.

When encountering a tactic at indentation `indent`:
* if the top of the stack is `IndentBullet (_, pre_indent)`,
  warn if `indent <= pre_indent`, focus the first goal
  and add `IndentTactic indent` of top of the stack
* otherwise, the top of the stack is `IndentTactic pre_indent`
  * if `indent = pre_indent`, do nothing
  * if `indent > pre_indent`, we have a *positive indentation*,
    * raise an error if there are not exactly two focused goals
    * then, focus the first goal and add `IndentTactic indent` of top of the stack
  * if `indent < pre_indent`, we have a *negative indentation*,
    * raise an error if there is any focused goal
    * then, repeatedly pop the stack until we get a focused goal on top of it
      (raise an error if it becomes empty)
    * warn if `indent` doesn't match the indentation on top of the stack
    * raise an error if there isn't exactly one focused goal
    * then, focus the first goal and add `IndentTactic indent` of top of the stack.

### Implementation details / notes

* The implementation will mostly happen in `proof_bullet.ml`, with
  some additional tracking of indentation in higher levels that can
  likely reuse current locations.
* The implementation will call `Proof_bullet.push` with a new
  `BulletIndent` type for each tactic.
* We'll need to add a `Proofview.nb_goals` (look at
  `Proofview.finished`).
* We'll need to add a `Proof.nb_focused_goals` (look at
  `Proof.no_focused_goal`).

## Running example

Previous example illustrating the state of the stack at each step:

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
(* stack = [] *)
split.
(* stack = [IndentTactic 0] *)
  idtac.
(* stack = [IdentTactic 2; IndentTactic 0] *)
  exact I.
idtac.
(* stack = [IndentTactic 0] *)
split; [|split].
-
(* stack = [IdentBullet (-, 0); IndentTactic 0] *)
  idtac.
(* stack = [IdentTactic 2; IdentBullet (-, 0); IndentTactic 0] *)
  exact I.
-
(* stack = [IdentBullet (-, 0); IndentTactic 0] *)
  idtac.
(* stack = [IdentTactic 2; IdentBullet (-, 0); IndentTactic 0] *)
  exact I.
idtac.
(* stack = [IndentTactic 0] *)
exact I.
Qed.
```

## More examples

### Warning and errors

```Coq
Set Default Goal Selector "!".
Set Bullet Behavior "Indent".

Goal True /\ True /\ (True /\ True).
Proof.
- (* Warning: useless bullet on single goal *)
Abort.

Goal True /\ True /\ (True /\ True).
Proof.
split.
-
  + (* Warning: useless bullet on single goal *)
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
split; [|split].  (* last subgoal, using negative indentation *)
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
