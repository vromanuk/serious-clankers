# Rust craft and control flow

## matklad — push ifs up, fors down

Source: [Push Ifs Up And Fors Down](https://matklad.github.io/2023/11/15/push-ifs-up-and-fors-down.html) (matklad, 2023).

**Idea:** Decide early (caller / outer function); leave the tight loop as straight-line work on already-valid data.

```rust
// Prefer: caller resolves Option / Result / mode, then calls a simple body
fn handle(req: Request) {
    let user = match req.user {
        Some(u) => u,
        None => return reject(),
    };
    process(user); // no nested ifs inside process for "missing user"
}

// Avoid: deep nesting inside the hot path for control decisions
```

When reviewing: flag functions that mix policy branching with dense loops when the branch could be lifted.

## Explicit over clever

Prefer **explicit** over clever; small composable pieces; errors you can see; avoid magic globals.

When reviewing:

- Hidden global config that changes behavior without parameters  
- “Clever” macros that obscure the decision  
- Prefer a boring `match` on a decision enum over side effects in a helper named `maybe_do_stuff`

## Density vs cleverness

Dense **local** code that states the domain is fine.  
Dense **abstraction** for one call site is not (small-surface rule).

## Formatter / linter

`rustfmt` + project clippy are the style enforcers. This file is for **shape of control flow and APIs**, not brace placement.
