# StochasticBatchedDP project (CLAUDE briefing)

This file orients a Claude Code session that opens this folder cold.

## What this project is

Stochastic Batched Dynamic Programming (SBDP): a scaling framework for joint
hardware-policy co-design of POMDPs at long horizons.  Standalone successor
to a research thread in the parent BFIMGaussian repo; targets its own paper.

**Read these in order when starting:**

1. `README.md` — framework overview, novelty, three ambition tiers, full 220-dim parametric `c` listed across 7 layers, baseline list, pseudocode.
2. `project_charter.md` — concrete experiment plan: φ_max=0.5, K_total=16-32, dim(c)=220, with **Higgins-Wiseman (geometric-Ramsey + adaptive feedback) as the primary baseline** (PCRB is secondary).  Five phases with go/no-go criteria.
3. `StochasticBatchedDP.tex` / `.pdf` — formal article.  §5 has the full step-by-step gradient derivation (n=2 detailed in §5.1-5.7, generalized n>2 detailed in §5.8 with Lemma 5.8.5 on score-function unbiasedness).

## Project context (carried over from prior sessions)

**Origin.**  This project was carved out of the BFIMGaussian repo where the original
joint-DP framework was developed (paper headline: 11.3× joint-DP/PCRB MSE at K=4 on a
superconducting-qubit flux sensor).  SBDP extends that framework past the K=4-5 exact-DP
ceiling by stochastic outer SGD with exact inner Bellman per batch.

**Why this configuration is impactful.**  At φ_max=0.5 (the wide-prior aliasing-floor
regime), the existing exact-DP joint-DP advantage collapses to 1.76× because four epochs
cannot disambiguate the multimodal posterior.  SBDP at K_total=16-32 should rescue the
advantage by enabling long-horizon disambiguation; this is the central testable claim.

**Baseline choice.**  Higgins-Wiseman (geometric-Ramsey with adaptive feedback phase) is
the canonical adaptive-QPE baseline (Berry-Wiseman 2000; Higgins et al. *Nature* 2007).
It is the only meaningful adaptive comparison; PCRB is non-adaptive and the win against
it at long horizon is unsurprising.  HW's failure mode at φ_max=0.5: its geometric
schedule starts at τ_max where the prior is ~1.85 fringes wide, wasting initial
measurements until τ becomes short enough to disambiguate.  SBDP-Bellman starts short to
disambiguate first, then long for refinement.

**Decision NOT to do pixel-based geometry.**  Considered and rejected.  Reasons: forward
model engineering would be multi-year (3D EM solver + quantization of distributed
circuits); per-pixel sensitivity is small (GHz wavelength is 1000× pixel scale); fab
doesn't accept arbitrary pixel layouts; marginal performance vs 220-dim parametric `c` is
likely <2×; story dilution risk.  Stick with parametric across 7 physics-meaningful
layers.

## User profile

Researcher in photonics / inverse-design AI.  H-index 32, ~8000 citations.  Pre-tenure at
Virginia Tech ECE (Bradley Department of Electrical and Computer Engineering).  Email:
`inversedesignai@gmail.com`.  Co-authors include Arvin Keshvari and William Tuxbury.

## Voice / style rules (persistent)

These are user-stated preferences carried over from the parent project:

- **No em-dashes** anywhere.  Prefer commas, parens, colons, semicolons.
- **Lean prose**: no purple metaphors, no meta-framing, no lab-notebook running commentary.
- **No "X, not Y" pattern**: prefer positive assertions.
- **Don't over-justify**: keep documents lean; don't preempt reviewers with defensive paragraphs.
- **Gloss jargon for physics readers** when relevant.
- **After pdflatex**: clean up `.aux/.log/.out/.toc/.bbl/.blg`; keep only `.tex` and `.pdf`.
- **Do not write planning/decision/analysis MD files unless explicitly asked.**

## Compute environment

- 380 cores available on the local machine (`/home/zlin`)
- GPU is available but not used for SBDP (CPU-only Bellman recursion)
- BFIMGaussian repo at `/home/zlin/BFIMGaussian` for reference (different repo; SBDP no longer lives there as of commit `5ef7244`)
- Persistent auto-memory at `~/.claude/projects/-home-zlin-StochasticBatchedDP/memory/`

## State at handoff

- Initial commit (`a45ae17` or successor) on `master`.
- New repo at `https://github.com/inversedesignai/StochasticBatchedDP` (public).
- No experimental modules implemented yet — Phase 0 of the project charter is the next step.
- The 1-day Phase 1 calibration test (K=8 SBDP at φ=0.5 with dim(c)=7 to verify the framework numerically) is the natural first compute experiment after Phase 0 infrastructure work.

## When in doubt

- The `.tex` is the technical reference; the README is the project overview; the project charter is the operational plan.
- If a reader asks about a derivation step, point them to the corresponding subsection in `StochasticBatchedDP.tex` §5.
- If a reader proposes adding pixel-based geometry, refer to the "Decision NOT to do pixel-based geometry" point above and the parametric-c layer list in README §"Tier 2".
