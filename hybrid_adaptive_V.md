# Terminal value design for SBDP

This note develops a principled choice of terminal value function V_T for
SBDP's batched receding-horizon Bellman recursion.  The result is a closed-form
hybrid of posterior variance and the recursive Bayesian Cramer-Rao bound, with
an optional adaptive residual on top.

## 1. Motivation: why V_T matters

SBDP solves the K_total-step adaptive measurement problem by decomposing it
into ⌈K_total / K_batch⌉ batches of K_batch steps each.  Within a batch, exact
Bellman recursion finds the optimal sub-policy.  Across batches, the receding
horizon advances: each batch starts from the posterior its predecessor left
behind.

The Bellman recursion within a batch needs a terminal condition.  At step
k_rem = 0 of the recursion (the last step of the batch, looking forward to
the boundary), the planner must assign a value to every reachable belief b at
the boundary.  This value V_T(b, k_rem) is the cost-to-go from b through the
remaining K_total − k_executed measurements.

If V_T were exactly the optimal cost-to-go V*(b, k_rem), the receding-horizon
decomposition would be globally optimal regardless of K_batch.  Any
approximation error in V_T propagates as suboptimality of the realized policy.
The size of this gap depends on K_batch (longer lookahead within a batch
forgives a worse V_T) and on how badly V_T misrepresents the future cost.

The design question is: what closed-form V_T tracks V*(b, k_rem) well enough
that K_batch = 4 is sufficient lookahead, and what residual correction can
adapt away the remaining gap?

## 2. Two natural anchors

Two closed-form candidates suggest themselves.  Each captures a different
aspect of the true V*.

### 2.1 Posterior variance (stop-now MSE)

V_canonical(b) = Var_{φ ~ b}[φ]

This is the MSE achieved if the agent stops measuring at b and reports the
posterior mean.  It has three appealing properties:

1. **Direct alignment with the objective.**  The Bellman backups carry the
   same units as the headline metric.  No proxy.
2. **Multimodality awareness.**  A wrapped multimodal posterior at φ_max = 0.5
   has large variance because of mode separation.  Using posterior variance as
   terminal value reads "multimodal posterior is expensive", pressuring the
   planner to disambiguate.  This is exactly the inductive bias SBDP exploits.
3. **Closed-form** for any explicit belief representation.

The flaw is structural: posterior variance assumes zero further measurements.
At a non-final batch boundary, k_rem more measurements are still scheduled,
and they will reduce the actual cost.  Posterior variance therefore
overestimates V*(b, k_rem) at every batch except the last.  The error is
largest at the first batch boundary (most measurements still to come) and
zero at the final batch boundary.

### 2.2 PCRB sum (best-case future)

V_canonical(b, k_rem) = sum of remaining Cramer-Rao contributions

Specifically, the recursive posterior CRB (Tichavsky-Muravchik-Nehorai, 1998)
seeded at the current belief b and propagated forward k_rem steps under the
optimal non-adaptive measurement schedule.

This anchor knows about the remaining budget.  At k_rem = 16 with
Heisenberg-scaled τ_j, the bound shrinks like 1/k_rem², matching the
asymptotic scaling of the true V*.

The flaw is the assumption of locally Gaussian curvature.  PCRB is built on
the Fisher information at the posterior mode and assumes the posterior shape
is well-approximated by a Gaussian centered there.  In the multimodal regime
this is wrong: the Fisher information at any single mode is finite and
optimistic, ignoring the cost of mode disambiguation.  PCRB therefore
underestimates V*(b, k_rem) in the wide-prior regime, exactly where SBDP must
make its hardest decisions.

### 2.3 Bias direction summary

| Anchor              | k_rem = 0       | k_rem large    | Multimodal regime |
|---------------------|-----------------|----------------|-------------------|
| posterior_variance  | exact           | overestimates  | overestimates     |
| PCRB_sum            | underestimates  | tight          | underestimates    |

True V* is sandwiched between them.  Neither alone tracks it across the
relevant range.

## 3. Why naive combinations fail

A first instinct is V_canonical = max(posterior_variance, PCRB_sum).  This
fails on two counts.

**Scale divergence with k_rem.**  Posterior variance is k_rem-independent
(it is a property of the current belief).  PCRB_sum decreases monotonically
with k_rem.  At any reasonable k_rem ≥ 1, posterior variance dominates PCRB
by one to three orders of magnitude.  The max collapses to posterior variance
for almost all (b, k_rem), discarding PCRB entirely.

**Numerical scale at φ_max = 0.5, K_total = 16:**

- posterior_variance at the prior: ≈ φ_max² / 3 ≈ 0.083 rad²
- PCRB_sum at k_rem = 16 with optimal non-adaptive Heisenberg schedule:
  on the order of 10⁻⁴ rad² or smaller
- ratio: 10² to 10³

A convex combination α(k_rem) · posterior_variance + (1 − α) · PCRB_sum has
the same problem dressed up.  The choice of α is arbitrary, and the two
terms live on systematically different scales for k_rem > 0.

The right combination is multiplicative through the Fisher-information
algebra, not additive in the variance domain.

## 4. The BCRB hybrid

Treat the current posterior b as the prior for a forward Bayesian CRB
recursion over the remaining k_rem steps.

Define cumulative Bayesian information

J_B(k_rem; b, c) = 1 / Var(b) + sum_{j = 1}^{k_rem} F_j(τ_j*, c),

where Var(b) is the variance of the current belief, F_j is the Fisher
information contribution of measurement j under hardware c at measurement time
τ_j, and τ_j* is the optimal non-adaptive choice (the τ that minimizes the
final bound).  The sum carries the contribution of the remaining measurement
budget; the prior term carries the information already in b.

Define the canonical anchor

V_canonical(b, k_rem; c) = 1 / J_B(k_rem; b, c).

### 4.1 Limit checks

- **k_rem = 0:** the sum is empty, J_B = 1 / Var(b), V_canonical = Var(b).
  Recovers posterior variance.  Tight at the final batch boundary.
- **k_rem large:** the Fisher sum dominates the prior term, V_canonical scales
  like 1 / Σ F_j, recovering the standard PCRB asymptotics.
- **All intermediate k_rem:** smoothly interpolated through one closed-form
  expression.

### 4.2 Why this is the right combination

The two anchors disagreed because they capture different terms of the same
underlying quantity.  Posterior variance is the inverse of the prior
information at the current belief.  PCRB_sum is the inverse of the future
Fisher information.  Bayesian CRB *adds the information contributions* before
inverting, which is the only operation that respects the way information
combines under repeated measurement.

Adding posterior_variance and PCRB_sum directly is wrong (information adds,
variances do not).  Inverting the *sum of inverses* is right.

V_canonical(b, k_rem; c) = (1 / Var(b) + Σ F_j)⁻¹

is the only formula in which:

- units agree across regimes (same MSE units throughout)
- the two pieces are dimensionally compatible by construction
- the limits in k_rem are correct
- multimodality enters naturally through Var(b)

### 4.3 Multimodality-awareness

A multimodal wrapped posterior at φ_max = 0.5 has Var(b) inflated by the
mode-separation contribution, even when the per-mode curvature is sharp.
This makes 1/Var(b) small, the prior term in J_B contributes little, and
V_canonical stays high.  The planner reads the boundary belief as expensive,
which is exactly the disambiguation pressure SBDP needs.

This is the property that PCRB_sum alone lacks: PCRB uses the Fisher
information at a single mode and ignores mode separation.  The BCRB hybrid
recovers it for free through the Var(b) prior term.

### 4.4 Naming

This object is the standard Tichavsky-Muravchik-Nehorai posterior CRB
recursion, seeded at the current belief b rather than at the global prior.
In some literatures this is called *running BCRB* or *online PCRB*.  Reusing
existing terminology avoids reviewer confusion.  The novelty in this note is
not the recursion itself (well-known since 1998), but its use as the
canonical terminal value in receding-horizon adaptive measurement.

### 4.5 Per-batch policy structure

The k_rem dependence of V_canonical changes the cross-batch policy structure
relative to a static V_T.

| Setting                      | Bellman operator         | Policy function           |
|------------------------------|--------------------------|---------------------------|
| Static V_T (constant)        | identical across batches | identical π*(b); only realized actions differ across batches because entry beliefs differ |
| Hybrid V (k_rem-dependent)   | batch-dependent          | different π*_k(b) per batch |

With the hybrid V, each batch's inner Bellman uses a different terminal
condition (V_canonical evaluated at that batch's boundary k_rem), so the
resulting policy function differs from batch to batch.  Mathematically this
is one non-stationary finite-horizon policy π*(b, k_rem) evaluated at the
appropriate k_rem within each batch.  ψ remains a single shared parameter
vector if the residual is included; the planner extracts different per-batch
policies from the same ψ.

**Implementation.**  Recompute the inner Bellman per batch.  The count-tuple
grid is anchored at each batch's entry belief regardless of V choice, so this
adds no overhead beyond the existing per-batch recursion.  Caching π*_k(b)
across batches is not useful because each k has its own k_rem.

## 5. Adaptive residual on top

V_canonical still assumes optimal *non-adaptive* future continuation.  The
true V* uses adaptive continuation and is bounded above by V_canonical.
Define a learned residual

V(b, k_rem; ψ, c) = V_canonical(b, k_rem; c) + δ(b, k_rem; ψ)

with the constraints

- δ(b, k_rem; ψ) ≤ 0  (adaptive cannot do worse than non-adaptive)
- δ(b, 0; ψ) = 0      (no residual at the final boundary, where V_canonical
                       is exact)

### 5.1 Parameterization

Choose belief features that encode the regimes where V_canonical is loosest:

- m₁(b) = 1 − mass in dominant posterior mode
- m₂(b) = smoothed mode count
- m₃(b) = entropy of b relative to wrapped Gaussian with the same first two
          moments
- m₄(b) = Var(b)

Plus k_rem as an explicit input.  Pass (m₁, m₂, m₃, m₄, k_rem) through a
small MLP or a polynomial of low total degree.  Output δ through a sigmoid
gated by V_canonical:

δ(b, k_rem; ψ) = − V_canonical(b, k_rem; c) · σ(MLP_ψ(features)) · (1 − exp(−k_rem))

The exp gate enforces δ → 0 as k_rem → 0; the sigmoid enforces the sign
constraint and bounds amplitude to V_canonical.  dim(ψ) ≤ 30 is plenty.

### 5.2 Update rule

Same K_batch-step TD bootstrap as before, but only the residual is fit:

L(ψ) = Σ_k [ δ(b_entry_k, k_rem_k; ψ)
            − ( R_k + V(b_exit_k, k_rem_k − K_batch; ψ_target, c)
                    − V_canonical(b_entry_k, k_rem_k; c) ) ]²
       + λ ||ψ||²

R_k is the realized batch return.  ψ_target is a periodic snapshot of ψ.
Setting λ → ∞ recovers the static BCRB hybrid as a clean ablation.

### 5.3 Decoupling from c-gradient

During the outer SGD step on c, treat ψ as fixed (semi-gradient).  The
envelope-theorem gradient on c then has two contributions: the within-batch
trajectory (handled by the existing SBDP machinery) and the boundary term

∂V/∂c at b_exit = ∂V_canonical/∂c + ∂δ/∂c.

∂V_canonical/∂c is closed-form through the Fisher recursion.  ∂δ/∂c is zero
because δ depends on c only through the belief features m_i(b), which feed
through the within-batch trajectory and are already accounted for there.

## 6. When the static hybrid suffices

The static hybrid V_canonical (ψ ≡ 0) is expected to suffice for the headline
SBDP results.  The residual δ is reserved for diagnostic regimes.  Four
arguments support this expectation.

**6.1 Receding horizon washes out the projection error.**

V_canonical assumes optimal non-adaptive future continuation, but this
assumption only governs the *projected* boundary value at the current batch.
The next batch reseeds the recursion at the realized exit belief, replacing
the projection with adaptive realization.  Cumulative non-adaptive bias is
bounded by the per-batch projection error, not by K_total.  The error does
not compound across batches.

**6.2 The inner Bellman absorbs adaptation locally.**

K_batch = 4 of exact Bellman is genuine adaptive lookahead.  Disambiguation
at φ_max = 0.5 (short τ first to break the 2π aliasing, long τ for refinement
once the posterior is unimodal) operates on a 1-3 step timescale.  K_batch = 4
captures the qualitative posterior transitions; the non-adaptive projection
is approximately correct once the posterior has gone unimodal, which happens
within one or two batches.

**6.3 Multimodality already enters through Var(b).**

The 1/Var(b) prior term inflates V_canonical exactly when the boundary belief
is multimodal.  This is the disambiguation pressure that δ would otherwise
have to learn.  The static hybrid gets it for free, no fitting required.

**6.4 Adaptive vs non-adaptive affects constants, not scaling.**

For the headline SBDP-vs-HW ratio (5× at K = 16, 10× at K = 32), the leading
scaling of the ratio is set by Heisenberg vs shot-noise-limited continuation,
which is the same for adaptive and non-adaptive optimal future schedules.
The adaptive correction at the boundary is a constant-factor improvement,
plausibly under 2× even at φ_max = 0.5.  The static hybrid likely lands
within that factor of the fully adaptive optimum, comfortably preserving
the order-of-magnitude headline.

### 6.5 When the residual becomes load-bearing

Three regimes where δ should be added back:

- **K_batch reduced to 2 or 1** (memory-constrained variant).  Less inner
  adaptive lookahead, more burden on the boundary value, larger projection
  error per batch.
- **Increment A underperforms expectations** AND ablation shows the gap is
  at the boundary value rather than outer-SGD optimization noise.  Diagnostic
  test: SBDP at K_total = 16 lands close to HW (ratio < 3×), and inflating
  K_batch to 8 closes the gap.  This isolates the deficit to lookahead.
- **Reviewer or co-author requests the fully adaptive version** as a
  robustness check.  Cheap to add as a follow-up; do not let it block
  Phase 1.

### 6.6 Decision rule

Run Phase 1 calibration with ψ ≡ 0.  Add δ only if Phase 1 underperforms
*and* the diagnostic above isolates the gap to the boundary value.  This
keeps the primary contribution legible: a closed-form receding-horizon
Bayesian-CRB anchor is a single-sentence theoretical statement.  The learned
residual is a one-paragraph extension if needed, omitted otherwise.

## 7. Recommended workflow

Three increments, each independently testable.

**Increment A (Phase 1 calibration).**  Static V_canonical only, ψ = 0.  This
is the closed-form BCRB hybrid.  No learning, no extra moving parts.  Use
this as the reference SBDP run.  If this already meets the headline targets
(K_total = 16: ≥ 5× HW; K_total = 32: ≥ 10× HW), the residual is unnecessary
and the writeup stays simple.  Per Section 6 this is the expected outcome.

**Increment B (Phase 3 if needed).**  Add δ with the parameterization above,
λ large (heavy regularization).  Goal: shave residual gap, demonstrate that
the static hybrid is already close to optimal.

**Increment C (research extension).**  Reduce λ, allow δ to do real work.
Compare to Increment A as ablation.  Report the SBDP-vs-HW ratio under each
of the three settings.

## 8. Implementation notes

**Computing Var(b).**  For the count-tuple belief representation, Var(b) is
the wrapped variance of the discrete posterior over φ.  Closed-form, O(N_φ)
per evaluation.

**Computing Σ F_j(τ_j*, c).**  Solve the non-adaptive PCRB optimization once
per outer SGD step.  This is a k_rem-dimensional optimization over the
schedule, but the objective is convex in the per-step Fisher contributions
and the optimal schedule has known closed form for the standard scqubit
forward model (geometric in τ for Heisenberg scaling, with corrections for
finite T_2).  Cache between SGD steps if c changes slowly.

**Differentiability through V_canonical.**  J_B is differentiable in c (each
F_j is differentiable in c).  The reciprocal V = 1/J_B is differentiable
wherever J_B > 0, which holds whenever Var(b) is finite or any F_j > 0.

**Stability of ψ updates.**  The sign and amplitude constraints on δ make
the residual bounded by V_canonical.  This prevents the runaway-target
pathology of unconstrained value learning.  Combined with target-network
ψ_target updates and an L2 prior, the ψ training is well-behaved.

## 9. Summary

The terminal value V_T at SBDP batch boundaries should be the recursive
Bayesian Cramer-Rao bound seeded at the current belief and propagated forward
under optimal non-adaptive future continuation:

V_canonical(b, k_rem; c) = ( 1 / Var(b) + Σ_{j=1}^{k_rem} F_j(τ_j*, c) )⁻¹.

This is the unique closed-form expression that combines the strengths of the
posterior-variance and PCRB-sum anchors without the scale and bias problems
of either alone.  It recovers posterior variance at the final batch boundary,
PCRB-sum asymptotics at long horizon, and is multimodality-aware through the
Var(b) term.

The static hybrid (ψ ≡ 0) is expected to suffice for the headline SBDP
results: receding horizon caps the projection error per batch, K_batch = 4
of inner Bellman absorbs local adaptation, Var(b) supplies multimodality
pressure for free, and adaptive vs non-adaptive at the boundary is a
constant-factor correction below the headline order of magnitude.  An
optional learned residual δ ≤ 0, gated by V_canonical and constrained to
vanish at k_rem = 0, is reserved for diagnostic regimes (K_batch ≤ 2,
isolated boundary-value gap from ablation).
