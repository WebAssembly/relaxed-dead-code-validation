# Relaxed Dead Code Validation

This proposal aims to make handling syntactically dead code simpler by relaxing its validation requirements.

Abstractly, dead code is no longer checked for violations of the [type system](https://webassembly.github.io/spec/core/valid/instructions.html#instructions).

As of the current proposal, dead code will still obey syntactic restrictions laid out in the [binary format](https://webassembly.github.io/spec/core/binary/instructions.html), including restrictions on the maximum size of immediates..

## Barebones Spec Changes

Extend the typing rules to the following form:
```
ot := t* | ⊥
st := ot -> ot

C ⊢ e* : st
```

Change the typing rules for `unreachable`, `br`, `br_table`, and `return` as follows:
```
C ⊢ unreachable : t* -> ⊥

C.labels[i] = t*
------------------------
C ⊢ br i : (t'*) t* -> ⊥

(C.labels[i] = t*)+
-----------------------------------
C ⊢ br_table i+ : (t'*) t* i32 -> ⊥

C.return = t*
--------------------------
C ⊢ return : (t'*) t* -> ⊥
```

Add the following typing rule:
```
C ⊢ e* : ⊥ -> t*
```

If the reference types proposal has already landed (likely), the addition of a bottom value type can be reverted (at least for now).

## Implementation Consequences

TODO
