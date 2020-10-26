# Reusing the Push-Pop Abstraction

Rather than having validation of each instruction be implemented differently when unreachable, most instructions can be validated independently of reachability by taking advantage of the push-pop abstraction.
At a high level, the abstraction is changed so that, when the stack is in the unreachable state, pushes and pop-with-expectations do nothing.
Instructions only need to reason specifically about reachability when they need to pop a type without any expectation of what it is.

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
  reachable : bool
}
```

#### Global Variables
Maintain the invariant that if `reachable` is `false` then `opds.size()` equals `ctrls[0].height`.
```javascript
var opds : opd_stack
var ctrls : ctrl_stack
var reachable : bool
```

#### Auxilliary Functions
Note the lack of an `Unknown` type in `pop`, and that `pop_opd(expect)` no longer returns a `val_type`.
```javascript
function push_opd(type : val_type) =
  if (reachable)
    opds.push(type)

function pop_opd() : val_type =
  error_if(opds.size() = ctrls[0].height)
  return opds.pop()

function pop_opd(expect : val_type) =
  if (reachable)
    error_if(pop_opd() =/= expect)

function push_opds(types : list(val_type)) = foreach (t in types) push_opd(t)
function pop_opds(types : list(val_type)) = foreach (t in reverse(types)) pop_opd(t)
```

```javascript
function push_ctrl(opcode : opcode, in : list(val_type), out : list(val_type)) =
  let frame = ctrl_frame(opcode, in, out, opds.size(), reachable)
  ctrls.push(frame)
  reachable := true
  push_opds(in)

function pop_ctrl() : ctrl_frame =
  error_if(ctrls.is_empty())
  let frame = ctrls[0]
  pop_opds(frame.end_types)
  error_if(opds.size() != frame.height)
  reachable := frame.reachable
  ctrls.pop()
  return frame

function label_types(frame : ctrl_frame) : list(val_types) =
  return (if frame.opcode == loop then frame.start_types else frame.end_types)

function unreachable() =
  opds.resize(ctrls[0].height)
  reachable := false
```

#### Validation of Opcode Sequences

Note that the implementation of reachable vs. unreachable instructions differs only for `drop` and `select` (though extensions might add more instructions that need such special casing).

```javascript
function validate(opcode) =
  switch (opcode)
    case (i32.add)
      pop_opd(I32)
      pop_opd(I32)
      push_opd(I32)

    case (drop)
      if (reachable)
        pop_opd()

    case (select)
      if (reachable)
        pop_opd(I32)
        let t = pop_opd()
        pop_opd(t)
        push_opd(t)

    case (unreachable)
      unreachable()

    case (block t1*->t2*)
      pop_opds([t1*])
      push_ctrl(block, [t1*], [t2*])

    case (loop t1*->t2*)
      pop_opds([t1*])
      push_ctrl(loop, [t1*], [t2*])

    case (if t1*->t2*)
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
      pop_opds(label_types(ctrls[n]))
      unreachable()

    case (br_if n)
      error_if(ctrls.size() < n)
      pop_opd(I32)
      pop_opds(label_types(ctrls[n]))
      push_opds(label_types(ctrls[n]))

    case (br_table n* m)
      error_if(ctrls.size() < m)
      for each (n in n*)
        error_if(ctrls.size() < n || label_types(ctrls[n]) != label_types(ctrls[m]))
      pop_opd(I32)
      pop_opds(label_types(ctrls[m]))
      unreachable()
```
