# LLVM IR Control-Flow Training

Hands-on exercise in how `br`, `icmp`, `phi`, `switch`, and basic blocks
represent branches, conditionals, and loops in LLVM IR using
`clang++`, `opt`, and `llvm-dis`/Graphviz to generate, inspect, and
visualize real IR and CFGs.

## Project layout

```
llvm-training/
├── README.md
├── src/
│   ├── condition.cpp        if/else example  -> check(int)
│   ├── loop.cpp              for-loop example -> sum(int)
├── ir/
│   ├── condition.ll         
│   ├── loop.ll              
│   ├── loop_noopt.ll         
│   ├── loop_mem2reg.ll       
│   ├── loop_O1.ll            
└── notes/
    ├── control-flow.md        *** the actual deliverable ***- annotated IR + answers
    └── images/                 CFG PNGs embedded in control-flow.md
```

**The deliverable is `notes/control-flow.md`.** Everything else in this
repo (`src/`, `ir/`) is the working material used to produce it.

## Requirements

- `clang++` (LLVM 18.x here)
- `opt`, `llvm-dis` (same LLVM install as clang++)
- `dot` (Graphviz), for rendering `.dot` → `.png`

macOS: `brew install llvm graphviz`, then put Homebrew's LLVM on your
`PATH` (Apple's system `clang` doesn't ship `opt`/`llvm-dis`):
```bash
export PATH="/opt/homebrew/opt/llvm/bin:$PATH"   # Apple Silicon
```

Check your setup:
```bash
clang++ --version
opt --version
dot -V
```

## Reproducing the IR

All three source files declare their functions `extern "C"` purely to
keep the IR's symbol names readable (`@check`, `@sum`, `@choose`
instead of C++'s mangled `@_Z5checki` etc.) it has no effect on the
control flow being studied.

**`check()` — if/else:**
```bash
clang++ -O0 -S -emit-llvm src/condition.cpp -o ir/condition.ll
```

**`sum()`- loop, requires an extra step to see `phi`:**
Plain `-O0` keeps every local in a stack slot (`alloca`/`load`/`store`)
and never emits a `phi` node at all. To actually observe the `phi`
nodes the exercise asks about, without over-optimizing the loop away
(which is what plain `-O1` does — see `ir/loop_O1.ll`), the `optnone`
attribute that `-O0` sets is disabled so `opt` is allowed to run just
the `mem2reg` pass (stack → register promotion) on its own:

```bash
clang++ -O0 -Xclang -disable-O0-optnone -S -emit-llvm src/loop.cpp -o ir/loop_noopt.ll
opt -passes=mem2reg -S ir/loop_noopt.ll -o ir/loop_mem2reg.ll
```


## Reproducing the CFG diagrams

```bash
cd ir
opt -passes=dot-cfg loop_mem2reg.ll -disable-output  
mv .sum.dot cfg.sum.dot
dot -Tpng cfg.sum.dot -o loop_cfg.png
```
Same pattern for `condition.ll` (writes `.check.dot`) and
`switch_example.ll` (writes `.choose.dot`).

> Older LLVM versions use the legacy pass-manager syntax instead:
> `opt -dot-cfg loop_mem2reg.ll` (no `-passes=`, no `-disable-output`).
> Run `opt --version` first if the command above errors out.

## What's answered where

`notes/control-flow.md` covers, in order:

- **Part 1 (`check`)**- annotated IR, then answers to: how `x > 5` is
  checked, how if/else is implemented, how LLVM picks the return value.
- **Part 2 (`sum`)**- why `-O0` alone shows no `phi`, the mem2reg IR
  annotated, then answers to: the `phi` node's role, how `i` is
  remembered across iterations, how the loop-exit condition works.
- **CFG diagrams**- embedded PNGs for both the loop and the if/else,
  with each block labeled by role (entry / header / body / latch / exit).
- **Summary**- a short recap of how `br`, `icmp`, and `phi` fit
  together, which the spec listed as optional.
