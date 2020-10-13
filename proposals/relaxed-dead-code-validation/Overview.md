# Relaxed Dead Code Validation

This proposal aims to make handling syntactically dead code simpler by relaxing its validation requirements.

At a high level, in dead code, any [type system](https://webassembly.github.io/spec/core/valid/instructions.html#instructions) constraint which depends on a pop from the type stack is skipped (including the pop itself). For example, `ref.is_null` in dead code will not perform a check to see if a nullable reference is at the top of the type stack.

Dead code will still obey syntactic restrictions laid out in the [binary format](https://webassembly.github.io/spec/core/binary/instructions.html), including restrictions on the maximum size of immediates. In addition, type stack-independent checks (such as checking that the immediate of a `local.get i` is within the bounds of the declared local variables) will still be carried out.

For reference, the current typing rules for instructions which result in subsequent "syntactically dead code" (`unreachable`, `br`, `br_table`, and `return`) are as follows:
```
C ⊢ unreachable : t* -> t_*

C.labels[i] = t*
------------------------
C ⊢ br i : (t'*) t* -> t_*

(C.labels[i] = t*)+
-----------------------------------
C ⊢ br_table i+ : (t'*) t* i32 -> t_*

C.return = t*
--------------------------
C ⊢ return : (t'*) t* -> t_*
```
where `t_*` is non-deterministic (in practice, angelically picked/deferred via an `Unknown` type).

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
C ⊢ e* : t* -> t'*
-------------------
C ⊢ e* : ⊥ -> ot
```

This final typing rule is somewhat fragile with post-MVP instructions. It's just for illustration and a more nuanced change to the formal typing rules may be appropriate.

## Implementation Consequences

TODO

At a high level, implementations no longer need to implement a polymorphic type stack. In dead code, pushes and pops are not carried out, and all dependent checks are elided. A one-off refactoring may be required of the current fused decode-validate logic.
