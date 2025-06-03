- Title: Named Notation Levels

- Drivers: Pierre Roux (@proux01)

----

# Named Notation Levels

## Declaring Levels

```
Declare Notation Level id [left associativity|right associativity|no associativity] [in custom_entry].
```

Each level is declared either left assoc (accepts left assoc and non
assoc notations) or right assoc (accepts right assoc and non assoc
notations) or non assoc (only accepts non assoc notations). Default:
no associativity.

Entry defaults to "in constr".

Levels are qualified. Declaring a level twice (in the same module) is
an error.

Each level has two characteristics, it is either:
* declared or not
* used or not.
When declared, a new level is declared but not used yet.

The `Declare Notation Level` command acts at Require time.

### Default Hardcoded Levels

* top (current 200, non assoc)
* app (current 10, left assoc)
* postfix (current 1, left assoc)
* bot (current 0, non assoc)

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

We enforce a total order between used levels. bot/top are always at
bottom/top respectively. However we maintain an (acyclic) graph of
constraints since position of a level can remain unspecified until its
first use.

```
Notation Level Constraint qualid ((=|<) qualid)+ [in custom_entry].
```

Entry defaults to "in constr".

Mentioning undeclared levels is an error.

An unsatisfiable contraint is an error.

For equality constraints, levels needs to be compatible (i.e. we
cannot require equality between left assoc and right assoc levels).

For backward compat, we have implicit constraints between numbered
levels (to be removed later when cleaning up numbered levels).

The `Notation Level Constraint` acts at Import time. Thus, to require
two different libraries with independent notation levels, one would:
* `Require Import` first library
* only `Require` the second one
* add the necessary `Notation Level Constraint`
* `Import` second library

### Corelib levels

Constraints in Init/Notations.v:
pow = 30 < inv = opp = 35 < mul = orb = 40 < add = andb = 50 < eq = 70
< lnot = 75 < land = 80 < lor = 85 < iff = 95 < arrow = 99

## Reserving Notations

Same as currently, except that named levels are allowed.

Mentioning an undeclared level when reserving a notation is an error
(for backward compat, only a warning for numbered levels).

All levels mentioned in the newly reserved notation become used (if
they weren't already). If the order of used levels is no longer total,
this is an error.

Associativity of the newly declared notation defaults to the
associativity of its level. It can only be changed to non assoc.

## Printing

```
Print Notation Levels.
```

would print all used levels as an ordered list, then declared but
unsuded levels and their constraints. All levels are printed along
with their associativity.
