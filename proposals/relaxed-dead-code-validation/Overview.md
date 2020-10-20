# Relaxed Dead Code Validation

This proposal aims to make handling syntactically dead code simpler by relaxing its validation requirements.

At a high level, in dead code, any [type system](https://webassembly.github.io/spec/core/valid/instructions.html#instructions) constraint which depends on a pop from the type stack is skipped (including the pop itself). For example, `ref.is_null` in dead code will not perform a check to see if a nullable reference is at the top of the type stack.

Dead code will still obey syntactic restrictions laid out in the [binary format](https://webassembly.github.io/spec/core/binary/instructions.html), including restrictions on the maximum size of immediates. In addition, type stack-independent checks (such as checking that the immediate of a `local.get i` is within the bounds of the declared local variables) will still be carried out.

For reference, the current typing rules for instructions which result in subsequent "syntactically dead code" (`unreachable`, `br`, `br_table`, and `return`) are as follows:
```
syntax:
ot := t*
st := ot -> ot

C ⊢ e* : st

rules:
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

Extend the typing syntax to the following form:
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

At a high level, implementations no longer need to implement a polymorphic type stack. In dead code, some validation checks are skipped: pushes and pops are not carried out, and all dependent checks are elided. A one-off refactoring may be required of the current fused decode-validate logic in order to switch to this modified algorithm (with reduced checks) when handling dead code.

### Validation Algorithm

What follows is a representation of the expected validation algorithm, in the style of https://webassembly.github.io/spec/core/appendix/algorithm.html. The original's caveats still apply: a real implementation will perform interleaved decode+validation.

#### Data Structures
Note the lack of an `Unknown` type.
```javascript
type val_type = I32 | I64 | F32 | F64

type opd_stack = stack(val_type)

type ctrl_stack = stack(ctrl_frame)
type ctrl_frame = {
  opcode : opcode
  start_types : list(val_type)
  end_types : list(val_type)
  height : nat
  unreachable : bool
}
```

### Global Variables
```javascript
var opds : opd_stack
var ctrls : ctrl_stack
```

### Auxilliary Functions
Note the lack of an `Unknown` type in `pop`.
```javascript
function push_opd(type : val_type) =
  opds.push(type)

function pop_opd() : val_type =
  error_if(opds.size() = ctrls[0].height)
  return opds.pop()

function pop_opd(expect : val_type) : val_type =
  let actual = pop_opd()
  error_if(actual =/= expect)
  return actual

function push_opds(types : list(val_type)) = foreach (t in types) push_opd(t)
function pop_opds(types : list(val_type)) = foreach (t in reverse(types)) pop_opd(t)
```

```javascript
function push_ctrl(opcode : opcode, in : list(val_type), out : list(val_type)) =
  let frame = ctrl_frame(opcode, in, out, opds.size(), false)
  ctrls.push(frame)
  push_opds(in)

function pop_ctrl() : ctrl_frame =
  error_if(ctrls.is_empty())
  let frame = ctrls[0]
  pop_opds(frame.end_types)
  error_if(opds.size() != frame.height)
  ctrls.pop()
  return frame

function label_types(frame : ctrl_frame) : list(val_types) =
  return (if frame.opcode == loop then frame.start_types else frame.end_types)

function unreachable() =
  opds.resize(ctrls[0].height)
  ctrls[0].unreachable := true
```

### Validation of Opcode Sequences

Note that no attempt is made to make this efficient or cleverly reduce line-count. "Reachability" could be a static parameter, or alternatively the reachability check could be pushed inside the pop/push functions in many cases.

```javascript
function validate(opcode) =
  let reachable = !ctrls[0].unreachable
  switch (opcode)
    case (i32.add)
      if (reachable)
        pop_opd(I32)
        pop_opd(I32)
        push_opd(I32)

    case (drop)
      if (reachable)
        pop_opd()

    case (select)
      if (reachable)
        pop_opd(I32)
        let t1 = pop_opd()
        let t2 = pop_opd(t1)
        push_opd(t2)

    case (unreachable)
      unreachable()

    case (block t1*->t2*)
      if (reachable)
        pop_opds([t1*])
      push_ctrl(block, [t1*], [t2*])

    case (loop t1*->t2*)
      if (reachable)
        pop_opds([t1*])
      push_ctrl(loop, [t1*], [t2*])

    case (if t1*->t2*)
      if (reachable)
        pop_opd(I32)
        pop_opds([t1*])
      push_ctrl(if, [t1*], [t2*])

    case (end)
      let frame = pop_ctrl()
      push_opds(frame.end_types)

    case (else)
      let frame = pop_ctrl()
      error_if(frame.opcode != if)
      push_ctrl(else, frame.start_types, frame.end_types)

    case (br n)
      error_if(ctrls.size() < n)
      if (reachable)
        pop_opds(label_types(ctrls[n]))
        unreachable()

    case (br_if n)
      error_if(ctrls.size() < n)
      if (reachable)
        pop_opd(I32)
        pop_opds(label_types(ctrls[n]))
        push_opds(label_types(ctrls[n]))

    case (br_table n* m)
      error_if(ctrls.size() < m)
      for each (n in n*)
        error_if(ctrls.size() < n || label_types(ctrls[n]) != label_types(ctrls[m]))
      if (reachable)
        pop_opd(I32)
        pop_opds(label_types(ctrls[m]))
        unreachable()
```
