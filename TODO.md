# vani-interval — TODO

> Compiler builtins that already exist and must NOT be reimplemented:
> `sin` `cos` `sqrt` `exp` `log` `abs` `floor` `ceil` `f64_pi()` `push`
> `pop` `len` `set` `vec`
>
> Depends on vani-calculus (`diff_central`) -- v0.1.0's only Kosh
> dependency, used solely by `ep_propagate_1var`. N3.1-N3.4 (interval
> arithmetic) are self-contained.

---

## v0.1.0 — Implemented ✓ (kosh-index ROADMAP.md item N3, N3.1-N3.6)

### N3.1 — Interval struct + core arithmetic (9 functions)
- [x] `iv_new`, `iv_from_point` -- construction
- [x] `iv_neg`, `iv_add`, `iv_sub` -- straightforward endpoint arithmetic
- [x] `iv_mul` -- all four lo/hi corner-pair products, min/max over them
      (operand signs affect which corner is extremal)
- [x] `iv_scalar_mul` -- handles a negative scalar flipping endpoint order
- [x] `iv_reciprocal`, `iv_div` -- caller-trust: divisor must not contain 0
      (division by a zero-spanning interval is unbounded)

### N3.2 — Interval elementary functions (6 functions)
- [x] `iv_sqrt`, `iv_exp`, `iv_log` -- monotonic, straight endpoint mapping
      (caller-trust on domain: `iv_sqrt`'s `lo >= 0`, `iv_log`'s `lo > 0`)
- [x] `iv_pow_i64` -- integer power via closed-form endpoint computation +
      sign case-split (odd n monotonic; even n needs a 3-way split since
      an interval spanning zero has its minimum AT zero, not an endpoint) --
      NOT naive repeated `iv_mul(a,a)`, so it's dependency-effect-free
- [x] `iv_sin`, `iv_cos` -- the critical-point case ("the fiddly part" per
      the roadmap): beyond the two endpoint values, checks whether a
      critical point (`crit_offset + k*pi`, value `(-1)^k`) falls inside
      `[lo,hi]` via a fixed 4-iteration window (`_iv_trig_range`, shared by
      both), keeping the loop bound a compile-time constant rather than a
      data-dependent one. Full-period intervals (`width >= 2*pi`) short-
      circuit to `[-1,1]` directly.

### N3.3 — Interval set operations (6 functions)
- [x] `iv_contains`, `iv_width`, `iv_midpoint`
- [x] `iv_is_empty`, `iv_intersect` -- intersect can come back empty
      (`lo > hi`), `iv_is_empty` is how a caller tells the two cases apart
- [x] `iv_union_hull` -- convex hull, not set union (covers the gap
      between disjoint intervals, same convention as a bounding box)

### N3.4 — Rigorous interval-bisection root-finding (1 function)
- [x] `iv_bisect_root` -- returns `Vec<Interval>` (every surviving
      candidate bracket), not a single `Interval`: a single-bracket
      narrowing algorithm is not actually rigorous once the interval
      arithmetic "dependency problem" is accounted for (an overestimated
      enclosure can make a root-free half look like it contains a root;
      discarding the other half then can throw the real root away). The
      list-based version only discards a half when it's *provably*
      root-free. Validated on `x^2-4` (dependency-free, converges to
      exactly one candidate) AND `x^3-x-2` (x appears twice, hits the
      dependency problem directly, still converges to the real root once
      the spurious candidate's overestimation shrinks enough to expose
      itself -- see `tests/test_rootfinding.vani`)

### N3.5 — First-order error propagation (4 functions, 1 private helper)
- [x] `ep_propagate_1var` -- single-variable, via vani-calculus's
      `diff_central`
- [x] `_ep_grad_fd` (private) -- central-difference gradient of a
      multivariate `f`, following vani-optimize's `fn(ref Vec<f64>, i64)
      -> f64` convention for multivariate function pointers without
      depending on vani-optimize (a few lines, same "not worth a
      dependency" call vani-vectorcalc made for its own differential
      operators)
- [x] `ep_propagate_independent` -- n independent variables,
      `sigma_f = sqrt(sum((df/dxi)^2 sigma_i^2))`
- [x] `ep_propagate_covariance` -- n variables with covariance,
      `sigma_f = sqrt(g^T Cov g)`, `Cov` flat row-major (vani-matrix
      convention)

### N3.6 — Closed-form propagation shortcuts (4 functions)
- [x] `ep_sum2`, `ep_sum_n` -- `sigma_f = sqrt(sum(sigma_i^2))`
- [x] `ep_product2`, `ep_quotient2` -- relative-error-in-quadrature form,
      scaled back to an absolute sigma

### Tests and examples
- [x] `tests/test_arithmetic.vani` -- N3.1 against hand-computed corner
      cases (including a mixed-sign multiplication)
- [x] `tests/test_elementary.vani` -- N3.2 against hand-computed values,
      including the full-period `iv_sin` short-circuit and the `iv_cos`
      critical-point case explicitly
- [x] `tests/test_setops.vani` -- N3.3, including an empty intersection
- [x] `tests/test_rootfinding.vani` -- N3.4 on both a dependency-free
      function and one that hits the dependency problem directly, plus a
      rigor check that the returned bracket's image still contains 0
- [x] `tests/test_error_propagation.vani` -- N3.5 (all three arities) and
      N3.6, including the composed N3.5-vs-N3.6 agreement check on
      `f(x1,x2)=x1*x2`
- [x] `examples/projectile_range_demo.vani` -- projectile range computed
      four ways (nominal, N3.5 1-var, N3.5 n-var, N3.1/N3.2 rigorous
      enclosure) plus N3.4 root-finding for a target range, tying every
      piece of the package together on one problem

### Safety annotations
- [x] `#[bounded_stack(bytes=N)]` on every function that does NOT take a
      function-pointer parameter, budgets set to `vanic check`'s exact
      reported worst-case (largest: `iv_sin`/`iv_cos` at 400/392 bytes,
      from the critical-point search)
- [x] Functions taking a function pointer (`iv_bisect_root`,
      `ep_propagate_1var`, `_ep_grad_fd`, `ep_propagate_independent`,
      `ep_propagate_covariance`) deliberately have NO `#[bounded_stack]`
      attribute -- the checker can't see through an indirect call to
      account for the callee's frame, so the true bound depends on the
      caller-supplied `f` and is documented in a comment instead
      (`bounded_stack = N + f's bounded_stack bytes`), same convention as
      vani-calculus's `bisect`/`jacobian_1d`/`hessian_diag`
- [x] No recursion anywhere in this library

---

## Future

No v0.2.0 is currently planned. Candidates if a concrete need shows up:
interval versions of `tan`/`atan`/hyperbolic functions (only sin/cos/sqrt/
exp/log/pow shipped, matching exactly what the roadmap itemized), a real
interval Newton method for `iv_bisect_root` (would converge faster than
bisection and can sometimes PROVE uniqueness of a root within a bracket,
unlike plain bisection -- needs an interval derivative, i.e. an
`fn(Interval) -> Interval` for f' as well as f), and a tighter interval
extension (e.g. centered/mean-value form) to reduce the dependency-problem
overestimation documented above and in `iv_bisect_root`'s doc comment.
