# vani-interval

Rigorous interval arithmetic and first-order error propagation for the
[vāṇी compiler](https://github.com/enthusiasticgeek/vani-compiler) —
kosh-index roadmap item N3.

Two distinct techniques share this package (per `kosh-index/ROADMAP.md`'s
N3 breakdown):

- **Interval arithmetic** (`iv_*`) — an `[lo, hi]` range provably
  containing the true value, self-contained, no dependency.
- **Error propagation** (`ep_*`) — the linearized
  `σ_f ≈ sqrt(Σ(∂f/∂xi)²σxi²)` formula science/engineering actually asks
  for day to day. Not rigorous, but reuses
  [vani-calculus](https://github.com/enthusiasticgeek/vani-calculus)'s
  `diff_central` for the single-variable case.

## Add to your project

```toml
# vani.toml
[deps]
interval = { registry = "kosh", version = "^0.1" }
```

```sh
vanic add interval
vanic build
```

## What's included (v0.1.0 — complete; see TODO.md)

| Module | Functions |
|---|---|
| N3.1 — core arithmetic | `iv_new`, `iv_from_point`, `iv_neg`, `iv_add`, `iv_sub`, `iv_mul`, `iv_scalar_mul`, `iv_reciprocal`, `iv_div` |
| N3.2 — elementary functions | `iv_sqrt`, `iv_exp`, `iv_log`, `iv_pow_i64`, `iv_sin`, `iv_cos` |
| N3.3 — set operations | `iv_contains`, `iv_width`, `iv_midpoint`, `iv_is_empty`, `iv_intersect`, `iv_union_hull` |
| N3.4 — root-finding | `iv_bisect_root` |
| N3.5 — error propagation | `ep_propagate_1var`, `ep_propagate_independent`, `ep_propagate_covariance` |
| N3.6 — closed-form shortcuts | `ep_sum2`, `ep_sum_n`, `ep_product2`, `ep_quotient2` |

## Interval arithmetic: the dependency problem

Naive interval arithmetic evaluates each *occurrence* of a variable
independently, not each variable once. `x - x` over `x = [1,3]` doesn't
evaluate to `[0,0]` — it evaluates to `[1,3] - [1,3] = [-2,2]`, because the
two occurrences aren't correlated. Any expression that uses its input more
than once (e.g. `x^3 - x - 2`, where `x` appears in both the cube and the
linear term) **overestimates** its true range. This is a textbook,
well-documented limitation of interval arithmetic, not a bug — enclosures
stay mathematically valid (never smaller than the truth), just looser than
optimal.

`iv_pow_i64` sidesteps this for integer powers specifically (it computes a
closed-form range from the endpoints and a sign case-split, not repeated
`iv_mul(a, a)`), but any caller-composed expression that reuses an interval
more than once still hits it — see `iv_bisect_root`'s doc comment and
`tests/test_rootfinding.vani`'s `iv_cubic` for a worked example where
`iv_bisect_root` still converges to the real root despite it.

## `iv_bisect_root`: why it returns `Vec<Interval>`, not `Interval`

A single-bracket bisection (keep narrowing left-or-right, discard the
other half) is **not** actually rigorous in the presence of the dependency
problem: at any step, an overestimated enclosure can make a genuinely
root-free half look like it might contain a root, and picking wrong
throws the real root away permanently.

`iv_bisect_root` instead tracks a *list* of surviving candidate brackets.
At each step, every non-narrow candidate splits in two, and BOTH halves
survive into the next round if their interval-evaluated image contains
zero. A half is only ever discarded when it's *provably* root-free (image
doesn't contain 0) — enclosures are never smaller than the truth, so this
never loses a real root. It usually converges back down to one candidate
(spurious ones get pruned once their width shrinks enough to expose the
overestimation — see the cubic example above), but can legitimately return
more than one for functions with multiple real roots in the bracket.

## Error propagation: `_ep_grad_fd`

`ep_propagate_independent`/`ep_propagate_covariance` need the gradient of
a multivariate `f`. vāṇी has no closures, so multivariate function
pointers follow vani-optimize's established `fn(ref Vec<f64>, i64) -> f64`
convention. `_ep_grad_fd` (package-private) computes that gradient via
central differences — a few lines, not worth a hard dependency on
vani-optimize, the same "not worth a dependency" call vani-vectorcalc made
for its own central-difference differential operators.

## Correctness

Beyond hand-computed closed-form checks in each `tests/*.vani` file, two
composed checks tie functions (and the two halves of this package)
together rather than testing them in isolation:

- **N3.5 vs. N3.6 agreement** — `ep_propagate_independent`'s general
  finite-difference path and `ep_product2`'s closed-form shortcut, run on
  the *same* `f(x1,x2) = x1*x2`, must produce the same σ_f (see
  `tests/test_error_propagation.vani`).
- **`examples/projectile_range_demo.vani`** — projectile range
  `R(v0,θ) = v0²sin(2θ)/g` computed four ways on one problem: nominal
  value, N3.5 single-variable propagation, N3.5 n-var propagation, a
  rigorous N3.1/N3.2 interval enclosure over the same measurement box (its
  width should be the same rough order of magnitude as `2σ_R` — the
  example prints both for comparison), and N3.4 interval bisection to
  rigorously bracket the launch angle hitting a target range.

## What this library does NOT provide

These are already vāṇी compiler builtins — call them directly, no import needed:

`sin` `cos` `sqrt` `exp` `log` `abs` `floor` `ceil` `f64_pi()` `push` `pop` `len` `set` `vec`

## License

MIT
