- Title: Proof scripts indentation checking

- Drivers: Pierre Roux (@proux01)

----

# Summary

We offer an optional checking of proof script indentation, through
warnings.

# Motivation

Bullets (typically `-`, `+`, `*`) were initially introduced by the
MathComp library. They were at that time mere comments, without any
semantic check by Coq. Checks were later implemented in Coq but with a
semantic dramatically incompatible with the established usage in
MathComp that also includes a strict indentation discipline. We thus
offer here an indentation checking that should be able to both catch
indentation mistakes and unexpected subgoals structure on already
indented scripts. Since nothing idnentation based can be consensual,
the mechanism is perfectly optional: it only emits warning, which can
either be silenced or turned as error for strict enforcement.

A typical limitation of the problematic bullet behavior is the
following kind of scripts, leading to ever increasing indentation:

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

## Examples

```Coq
(* script without warning *)
Goal True /\ True /\ (True /\ True).
Proof.
split.
  exact I.  (* positive indentation, exactly two goals *)
split; [|split].  (* last subgoal, using negative indentation *)
- exact I. (* regular use of bullets *)
- exact I.
exact I.  (* last subgoal, using negative indentation *)
Qed.
```

```Coq
(* script without warning *)
Goal True /\ True /\ (True /\ True).
Proof.
split.
-
   exact I.
-   split; [|split].
    + exact I.
    +     exact I.
    +   exact I.
Qed.
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
exact I.  (* Warning: expected indentation or bullet *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
-
   exact I.
-   split; [|split].
    + exact I.
    exact I.  (* Warning: expected bullet + *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
-
   exact I.
-   split; [|split].
    + exact I.
    * exact I.  (* Warning: expected bullet + *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
-
   exact I.
-   split; [|split].
    + exact I.
    +     exact I.
       exact I.  (* Warning: expected indentation at column 4 or bullet + *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
  split.
exact I.  (* Warning: expected indentation or bullet *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
- split.  (* Warning: ignoring unexpected bullet *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
- 
  -  (* Warning: ignoring unexpected bullet *)
```

```Coq
(* script without warning *)
Goal True /\ True /\ (True /\ True).
Proof.
split.
-    idtac.
   exact I.  (* Warning: expected identation at column  5 *)
```

```Coq
(* script without warning *)
Goal True /\ True /\ (True /\ True).
Proof.
split.
-    idtac.
       exact I.  (* Warning: expected identation at column  5 *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
- exact I.
   - (* Warning: expected indentation at column 0 *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
  - exact I.
- (* Warning: expected indentation at column 2 *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
- exact I.
- split; [|split].
+  (* Warning: expected indentation at column 2 *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
- exact I.
- split; [|split].
    +  (* Warning: expected indentation at column 2 *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
- exact I.
- split; [|split].
  -  (* Warning: bullet - is already used *)
```

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
split.
-
exact I.  (* Warning: expected indentation at least at column 1 *)
```

## Algorithm

We use a stack of following type.

```ocaml
type column = int  (* non negative *)
type indent =
  | Bullet of bullet_type * column
  | Tactic of column
type remaining = int  (* non negative *)
type stack = list (indent * remaining)
(* if not empty, bottom element of the stack must be of the form `(Tactic _, 1)` *)
(*
* empty stack means beginning of the proof, waiting for first tactic
* stack `(Bullet _, _) :: stk` means we are waiting for the first
  tactic handling a bullet
* stack `(Tactic _, _) :: (Bullet _, rem) :: stk` means we are
  handling a bullet subgoal, with `rem` bullets remaining (after the
  current one)
* stack `(Tactic _, _) :: (Tactic _, _) :: stk` means we are currently in a first
  subgoal of two, with positive indentation
*)
```

Each bullet and tactic is considered sequentially as a corresponding
`indent`, with `nb_goals >= 1` being the current number of goals
(before executing the considered tactic):
* If the `indent` is `Bullet (bt, c)`:
  * If the stack is empty or of the form `(Bullet _, _) :: _`, then
    issue `Warning: ignoring unexpected bullet`.
  * If the stack is of the form `(Tactic c', rem) :: stk`
    * If `nb_goals = rem`, then issue `Warning: ignoring unexpected bullet`.
    * If `nb_goals > rem` (starting new bullet sequence):
      * If `bt` appears in `stk`, then issue `Warning: bullet <bt> is already used`.
      * If `c <> c'`, then issue `Warning: expected indentation at column <c'>`.
      * Then add `(Bullet (bt, c), nb_goals - rem)` on top of the stack.
    * Otherwise `nb_goals < rem`:
      * If `stk` is empty or of the form `(Tactic _, _) :: _`,
        then issue `Warning: ignoring unexpected bullet`.
      * Otherwise `stk` is of the form `(Bullet (bt', c'), rem') :: stk'`:
        * If `rem' > 0` (next bullet of the `bt'` sequence):
          * If `bt <> bt'`, then issue `Warning: expected bullet <bt'>`.
          * If `c <> c'`, then issue `Warning: expected indentation at column <c'>`.
          * Then, replace the stack with `(Bullet (bt, c), rem' - 1) :: stk'`.
        * Otherwise `rem' <= 0` (previous bullet was the last one of
          its sequence): replace the stack with `stk'` and reconsider
          `Bullet (bt, c)`.
* Otherwise, the `indent` is `Tactic c`:
  * If the stack is empty (first tactic of the script), then replace
    the stack with `[(Tactic c, 1)]`.
  * If the stack is of the form `(Bullet (_, c'), _) :: stk` (first
    tactic of a bullet):
    * if `c <= c'`, then issue `Warning: expected indentation at least
      at column <c'+1>`.
    * Then add `(Tactic c, nb_goals)` on top of the stack.
  * Otherwise, the stack is of the form `(Tactic c', rem) :: stk`:
    * If `nb_goals = rem` (next tactic of a sequence of tactics):
      * If `c <> c'`, then issue `Warning: expected indentation at
        column <c'>`.
      * Then replace the stack with `(Tactic c, rem) :: stk`.
    * If `nb_goals > rem` (potentially starting a subgoal with
      positive indentation):
      * If `nb_goals > rem + 1`, then issue `Warning: expected bullet`.
      * If `c <= c'`, then issue `Warning: expected indentation or bullet`.
      * Then add `(Tactic c, nb_goals)` on top of the stack.
    * Otherwise `nb_goals < rem` (potentially negative indentation for
      a last subgoal):
      * `stk` cannot be empty since this would mean `rem = 1 <=
        nb_goals` according to the type invariant of `stack`.
      * If `stk` is of the form `(Tactic c'', _) :: stk'` (second
        subgoal after a positive indentation):
        * If `c <> c''`, the issue `Warning: expected indentation at
          column <c''>`
        * Then, replace stack with `(Tactic c, nb_goals) :: stk'`.
      * Otherwise `stk` is of the form `(Bullet (bt, c''), rem') ::
        stk'`
        * if `rem' <= 0` (previous bullet was the last one of its
          sequence): replace the stack with `stk'` and reconsider
          `Tactic c`.
        * Otherwise `rem' > 0` (potentially last goal of a bullet
          sequence, with negative indentation):
          * If `rem' > 1`, then issue `Warning: expected bullet <bt>`.
          * If `c <> c''`, then issue `Warning: expected indentation
            at column <c''>`.
          * Then, replace the stack with `stk'`.

### Implementation details / notes

* The implementation could mostly happen in `proof_bullet.ml`, with
  some additional tracking of indentation in higher levels that can
  likely reuse current locations.
* The implementation should call `Proof_bullet.push` with a new
  `Tactic _` value for each tactic.
* We'll need to add a `Proofview.nb_goals` (look at
  `Proofview.finished` or `Proof.no_focused_goal`).

### Running example

Previous example illustrating the state of the stack at each step:

```Coq
Goal True /\ True /\ (True /\ True).
Proof.
(* stack = [] *)
split.
(* stack = [(Tactic 0, 1)] *)
  idtac.
(* stack = [(Tactic 2, 2); (Tactic 0, 1)] *)
  exact I.
idtac.
(* stack = [(Tactic 0, 1)] *)
split; [|split].
-
(* stack = [(Bullet (-, 0), 2); (Tactic 0, 1)] *)
  idtac.
(* stack = [(Tactic 2, 3); (Bullet (-, 0), 2); (Tactic 0, 1)] *)
  exact I.
-
(* stack = [(Bullet (-, 0), 1); (Tactic 0, 1)] *)
  idtac.
(* stack = [(Tactic 2, 2); (Bullet (-, 0), 1); (Tactic 0, 1)] *)
  exact I.
idtac.
(* stack = [(Tactic 0, 1)] *)
exact I.
Qed.
```
