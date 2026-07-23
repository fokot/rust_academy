# Closures & Iterators

## Closures are structs that capture their environment
Which trait a closure implements is inferred from **how the body uses captures** — you don't choose it.

| Trait | Uses captures | Callable | Call takes |
|---|---|---|---|
| `Fn` | reads (`&`) | many times | `&self` |
| `FnMut` | mutates (`&mut`) | many times | `&mut self` |
| `FnOnce` | consumes (moves a capture out) | **once** | `self` (by value) |

Nesting: `Fn` ⊂ `FnMut` ⊂ `FnOnce`. A bound accepts its own trait and stricter ones below it:
- `F: FnOnce` accepts everything (loosest requirement).
- `F: FnMut` accepts `Fn`, `FnMut`.
- `F: Fn` accepts only `Fn` (pickiest).

`move` is **orthogonal**: it controls capture *by-value vs by-ref*, NOT which `Fn*` trait. A `move` closure that only reads is still `Fn`.

```rust
let s = String::from("hi");
let c = move || { drop(s); };   // consumes s → FnOnce, one-shot
// let c = move || println!("{s}");  // only reads → Fn, callable many times
```

Calling borrows the closure the same way its body uses the environment → an `FnMut` closure needs a `mut` binding:
```rust
let mut count = 0;
let mut inc = || { count += 1; };   // FnMut — note `mut inc`
inc(); inc();
println!("{count}");                // 2 — borrow ended at last inc() (NLL)
```
To call an `FnMut` param, it too must be `mut`:
```rust
fn call_twice<F: FnMut()>(mut f: F) { f(); f(); }
```

## Iterators: iter / iter_mut / into_iter is an ownership choice
```rust
v.iter()        // &T      — borrows v, v survives
v.iter_mut()    // &mut T  — exclusive borrow (needs `mut v`)
v.into_iter()   // T       — CONSUMES v, v is gone afterwards
```
`for x in v` desugars to `v.into_iter()` (consumes); `for x in &v` → `v.iter()` (borrows).
Same shared/exclusive/owned trichotomy as `&`/`&mut`/move, applied to a collection.

## Always lazy
Adapters (`map`, `filter`, …) do nothing until a **consumer** runs them: `for`, `.collect()`, `.sum()`, `.fold()`, `.count()`.
```rust
v.iter().map(|x| println!("{x}"));   // prints NOTHING — no consumer (warns: unused Map)
for x in &v { println!("{x}"); }     // side effects → use a for loop, not map
```
`collect` needs a target type: `.collect::<Vec<_>>()` or `let v: Vec<_> = it.collect();`.

## Gotchas (from the dojo)
- Can't mutate a collection while iterating it — `for x in &v { v.push(*x); }` = shared+exclusive borrow overlap (iterator invalidation, a compile error here). Fix: snapshot/clone first, or index with a length snapshot.
- `x` from `iter()` is `&T`; arithmetic auto-derefs (`x * 2` works on `&i32`).
- Use `into_iter()` only when you want to consume the collection (e.g. move `String`s out without cloning).

## vs Scala
- Iterators are lazy like Scala `Iterator`/`View`, never eager like `List.map`. Forgetting the consumer is silent, not eager.
- `FnOnce` has no Scala analog — GC makes "consumes its capture, callable once" a non-issue there.
- `iter`/`iter_mut`/`into_iter` is an ownership decision Scala never forces (everything is a GC reference).
