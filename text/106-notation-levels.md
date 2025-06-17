- Title: Named Notation Levels

- Drivers: Pierre Roux (@proux01)

----

# Named Notation Levels

## Declaring Levels

```
Declare Notation Level id [left associativity|right associativity|no associativity] [in custom_entry].
```

Each level is declared either left assoc or right assoc or non
assoc. Default: no associativity.

Entry defaults to "in constr".

Levels are qualified. Declaring a level twice (in the same module) is
an error.

The `Declare Notation Level` command acts at `Import` time.

### Default Hardcoded Levels

* bot (current 200, non assoc)
* app (current 10, left assoc)
* postfix (current 1, left assoc)
* top (current 0, non assoc)

Question: should we use other names for bot/top: base and closed for instance?

### Corelib levels

We have some levels declared in Init/Notations.v:
* Corelib.Init.Notations.arrow (current 99, right assoc, e.g., "x -> y")
* Corelib.Init.Notations.iff (current 95, non assoc, e.g., "x <-> y")
* Corelib.Init.Notations.lor (current 85, right assoc, e.g., "x \/ y")
* Corelib.Init.Notations.land (current 80, right assoc, e.g., "x /\ y")
* Corelib.Init.Notations.lnot (current 75, right assoc, e.g., "~ x")
* Corelib.Init.Notations.eq (current 70, non assoc, e.g., "x = y", "x <= y", "x < y")
* Corelib.Init.Notations.add (current 50, left assoc, e.g., "x + y")
* Corelib.Init.Notations.mul (current 40, left assoc, e.g., "x * y")
* Corelib.Init.Notations.opp (current 35, right assoc, e.g., "- x")
* Corelib.Init.Notations.inv (current 35, right assoc, e.g., "/ x")
* Corelib.Init.Notations.pow (current 30, right assoc, e.g., "/ y")
* Corelib.Init.Notations.andb (current 50, left assoc, e.g., "x && y")
* Corelib.Init.Notations.orb (current 40, left assoc, e.g., "x || y")

## Ordering Levels

Bot/top are always at bottom/top respectively. We maintain an acyclic
graph of constraints.

```
Notation Level Constraint qualid ((=|<) qualid)+ [in custom_entry].
```

Entry defaults to "in constr".

Mentioning undeclared levels is an error.

An unsatisfiable contraint (i.e., making the graph cyclic) is an error.

For equality constraints, levels needs to be compatible (i.e. have the
exact same associativity).

For backward compat, we have implicit constraints between numbered
levels (to be removed later when cleaning up numbered levels).

Question: should we warn when adding an already there (potentially by
transitivity) constraint.

The `Notation Level Constraint` acts at `Import` time.

### Removing constraints

In could happen that two libraries A and B impose incompatible orders
on two common dependencies D and E (for instance A would constrain
`D.l < E.l` and B would constrain `E.l < D.l`). In order to be able to
use A and B together, we need a way to remove constraints (for
instance remove the `D.l < E.l` constraint after loading A, in order
to be able to load B).

```
#[remove] Notation Level Constraint qualid ((=|<) qualid)+ [in custom_entry].
```

Attempting to remove a non existing constraint is a warning (error by
default).

Note: when removing an equality constraint, we need to trigger the
"common prefix with incompatible levels" warning, in case the
no-longer equal levels are used in common prefixes of reserved
notations.

Question: should we warn when a removed constraint remains there by
transitivity?

### Corelib levels

Constraints in Init/Notations.v:
arrow = 99 < iff = 95 < lor = 85 < land = 80 < lnot = 75
< eq = 70 < add = andb = 50 < mul = orb = 40 < inv = opp = 35 < pow = 30

## Reserving Notations

Same as currently, except that named levels are allowed.

Mentioning an undeclared level when reserving a notation is an error
(for backward compat, only a warning for numbered levels).

Associativity of the newly declared notation is the associativity of
its level. It cannot be changed.

Reserving a notation twice with different levels remains an error.

Reserving a notation with a prefix common to an already reserved
notation, but with different levels, remains a warning ("common prefix
with incompatible levels").

## Printing

```
Print Notation Levels [in custom_entry].
```

would print all used levels as an ordered list, then declared but
unsuded levels and their constraints. All levels are printed along
with their associativity.

Entry defaults to "in constr".

The `Print Grammar` command should also print the subset of levels
used in the printed grammar.

## Remarks

* We need to modify our camlp5 fork (file gramlib/grammar.ml) to work
  with non total ordering of levels.
* Ignoring the "common prefix with incompatible levels" warning could
  lead to unexpected results (essentially, reserved notations "not
  working"), just as is already the case today.
* Attempting to use notations at incomparable levels will yield an
  error, requiring either parenthesizing or ordering the levels. E.g.,
  if `_ #A _` is defined in some lib A at some level and `_ #B _` is
  defined in some lib B at another --- incomparable --- level, then we
  don't know how `x #A y #B z` should be parsed (is it `x #A (y #B z)`
  or `(x #A y) #B z`?) and this yields an error at parsing time.
