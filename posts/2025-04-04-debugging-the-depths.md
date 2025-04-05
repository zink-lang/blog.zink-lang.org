---
author: "@g4titan"
twitter: "g4titanx"
date: "2025-04-04"
description: "Solving Stack Mismatches in Zink's ERC20 Implementation"
labels: ["zink", "rust", "evm"]
title: "Debugging the Depths"
---

I’ve been tangled up in a bug in Zink lately that’s been equal parts maddening and fascinating. It’s [Issue #306](https://github.com/zink-lang/zink/issues/306): the ERC20 transfer test in `zink/examples/erc20.rs` fails with an "Insufficient balance" error, even though the storage shows the right balance. What started as a contract-level puzzle quickly spiraled into a compiler-level labyrinth, which led to [Issue #324](https://github.com/zink-lang/zink/issues/324) about stabilizing stack checks. This post is my raw dump—everything I’ve pieced together so far, technical and messy, a map I can always retrace.

Tianyi encouraged me to write this, pointing out that it’s not just about solving the bug, it’s about building a mental framework for tackling hairy problems like this. He’s right, this might be one of the toughest nuts I’ve cracked in my career, and even if I stumble into a fix with a few lucky lines, the process matters. So here’s the story: the problem, the suspects, the debugging steps, the fixes I’ve tried, and where I’m stuck.

## The Obvious Problem: ERC20 Transfer Goes Haywire

As stated in [Issue #306](https://github.com/zink-lang/zink/issues/306), the `ERC20` transfer test fails with "Insufficient balance" despite storage showing correct balance value. Yet, during the transfer execution, it behaved as if the sender didn’t have enough tokens (broke, but that wasn't the case)

```rust
evm = evm.commit(false);
let info = evm
    .calldata(&contract.encode(&[
        b"transfer(address,uint256)".to_vec(),
        spender.to_bytes32().to_vec(),
        half_value.to_bytes32().to_vec(),
    ])?)
    .call(address)?;
println!("{info:?}");
assert_eq!(info.ret, true.to_bytes32(), "{info:?}");
```

## Debugging: Chasing the Stack Ghost

In an attempt to trace this issue, I added debug prints at some points in the test and in the process of doing this I noticed a bug in `_update`'s logic:
```rust
if to.eq(Address::empty()) {
    TotalSupply::set(TotalSupply::get().sub(value));
} else {
    TotalSupply::set(TotalSupply::get().add(value));
}
```
Now, this incorrectly updates TotalSupply instead of the recipient’s balance. It’s not the cause of the current revert (since that happens on the sender’s check), but it’ll break the transfer’s correctness. so I updated it:
```rust
let to_balance = Balances::get(to);
Balances::set(to, to_balance.add(value));
```

Back to debugging, after adding custom logs at several points in the test, I was able to track the source of the issue. The issue orginates from` _update`’s else branch:
```rust
if from.eq(Address::empty()) {
    TotalSupply::set(TotalSupply::get().add(value));
} else {
    DebugEvent::Balance(U256::from(42)).emit();
}
DebugEvent::TestLog(U256::from(99)).emit();
```

The bug wasn’t “Insufficient balance” (that was a misread)—it was a stack underflow when `_update` returned to `_transfer`. `_transfer` expects a bool (`SP = 1`), but `_update` was leaving `SP = 0`, tanking at `call_return`’s `_jump()`. So I traced it to the else branch not setting up a return value, unlike linear flows that implicitly worked (straight up logic without if-else, just if branch worked too).

I fixed it by updating `call_return` to push 1 for `empty-result` internal calls:
```rust
pub fn call_return(&mut self, results: &[ValType]) -> Result<()> {
    let len = results.len() as u16;
    if results.is_empty() {
        if self.sp() == 0 {
            self._push1()?; // Push 1 for _transfer
        }
        self._jump()?;
    } else {
        while self.sp() > len + 1 {
            self._drop()?;
        }
        self.shift_stack(len, false)?;
        self._jump()?;
    }
    Ok(())
}
```

This fixed the stack issue—`_update` now returns true to `_transfer`, and the underflow error is gone. But then that triggered public functions like `name()` started failing with `InvalidJump`. Turns out, `handle_frame_popping`’s new version I added was the cause:
```rust
_ => {
    self.table.label(frame.original_pc_offset, self.masm.pc());
    self.masm._jumpdest()?;
    if !self.is_main && self.abi.is_none() && self.ty.results().is_empty() && self.masm.sp() == 0 {
        self.masm._push1()?;
        self.masm._jump()?;
    }
    Ok(())
}
```

The `_push1(1)` and `_jump()` worked for `_update`, but for `name()` (a public call with `abi.is_some()`), It added an extra jump that broke the `main_return` flow (`RETURN` for string data). Reverting to the old version fixed `name()`:
```rust
_ => {
    self.table.label(frame.original_pc_offset, self.masm.pc());
    self.masm._jumpdest()?;
    Ok(())
}
```
Now `name()` returns `"The Zink Language"` again, but `_transfer`’s hitting `InvalidJump (ret: [] instead of [..., 1] at erc20.rs:295:9)`. The stack’s fine `(SP = 1 from call_return)`, but the jump target’s off `call_return`’s `_jump()` isn’t landing at `_transfer`’s return point. The jump table’s `original_pc_offset` isn’t syncing right I suppose.

I asked Tianyi about this and he said "...it could be caused by our mock of stack usage in compilation". so he opened [Issue #324](https://github.com/zink-lang/zink/issues/324) and decided that we tackle that first. He also pointed to Huff’s dispatching docs—stack outputs like takes (1) returns (1)—and suggested tests. In response to this, I wrote `tests/stack.rs` for `if-else`, `loops`, and `calls`.

## Fixes So Far

Here’s what I’ve tried to stabilize the stack:

- Fix 1: Tweak Internal Calls
In visitor/call.rs, I cut the initial increment_sp(1) in call_internal and adjusted SP post-call:
```rust
let current_sp = self.masm.sp();
while current_sp > *results as u16 {
    self.masm._drop()?;
}
if current_sp < *results as u16 {
    self.masm.increment_sp(*results as u16 - current_sp)?;
}
```

- Fix 2: Enforce Stack at _end
In visitor/control.rs, I made _end check SP against returns:

```rust
if self.masm.sp() != self.returns {
    return Err(Error::StackMismatch { expected: self.returns, found: self.masm.sp() });
}
```
This caught mismatches but didn’t fix the root cause—SP was still off.

Right now, the tests fail with:

- `dispatcher_stack: SP = 2, expected 1` errprs . Internal calls leave a ghost item.
- `if_else_stack: Extra function with SP = 1, expected 0`. Compiler artifact?
- `loop_stack: Returns 8, not 7.` Test runs the wrong WAT.

Apparently, the stack’s still haunted 😂. I suspect `call_internal`’s jump handling or `function::new`’s setup is the culprit. 
Anyways, I'm glad I took the advice to write this, it’s clearing my head, but I’m not out of tricks yet.

## My View: Stack Management is a Puzzle
The EVM’s stack is unforgiving, and Zink’s job is to map Rust’s abstractions onto it perfectly. Every push and pop has to align, or you’re toast. I’m starting to see stack management as a puzzle, to be honest, each function’s a piece. It’s brutal, but I’m hooked, solving this is an important breakthrough for Zink.
