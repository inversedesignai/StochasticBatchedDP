# Candidate applications for SBDP

Problems where SBDP's specific niche pays off: multimodal or aliased posterior
that benefits from short-horizon disambiguation before long-horizon refinement,
high-dimensional differentiable hardware parameters, long horizon required for
near-Heisenberg scaling, existing adaptive baseline that is suboptimal in the
wide-prior regime.

## Strongest fits

### 1. NV-center magnetometry with adaptive Ramsey

Direct physics analog of the superconducting flux qubit: identical Ramsey-fringe
aliasing, identical Higgins-Wiseman canonical baseline, identical short-then-long
advantage at wide prior.  The hardware vector is richer than the scqubit case:

- NV creation parameters (implantation depth, density, isotopic purity)
- Microwave pulse library (composite π/2 pulses, dynamical-decoupling shapes)
- Photon-collection optics (immersion lens, parabolic reflector, SIL geometry)
- Readout schedule (single-shot vs averaged, gated photon counting)

Broader audience than scqubit: biomedical imaging (single-cell magnetometry,
neural current sensing), geophysics (magnetic anomaly detection), materials
characterization.  Forward models exist (e.g., quTip + ray-tracing for the
optics) and are differentiable with modest effort.

Risk: framed narrowly this looks like "another QPE win".  Pitch the
optics-and-pulse co-design as the new contribution; the magnetometry community
already accepts adaptive Ramsey as the right comparison.

### 2. Atom-interferometer gravimetry and optical-clock interrogation

Same Ramsey aliasing physics with a different hardware stack.  The hardware
vector includes:

- Laser pulse shaping (Raman/Bragg pulse profile, chirp parameters)
- Beam-splitter timing and atom-optics geometry
- Trap parameters (waist, detuning, lattice depth)
- Magnetic and acoustic shield design

Long horizon is forced by coherence times in the seconds-to-minutes range.
Multimodal aliasing in the 2π-wrapped accumulated phase is the same structure as
the flux case.  Commercial relevance: inertial navigation (GPS-denied
positioning), gravity surveying for resource exploration, geodesy.  Reviewer
pool is largely disjoint from the superconducting-qubit audience, which doubles
the impact surface for one paper.

### 3. Hamiltonian learning with experiment co-design

Adaptive Bayesian inference of many-body Hamiltonian parameters where each
epoch chooses a measurement basis and an evolution time.  Multimodality of the
posterior comes from spectrum aliasing, accidental near-degeneracies, and
symmetry-related parameter ambiguity.  Hardware DOFs are the device used to
synthesize the evolution: coupler geometry, drive-line filters, readout
resonator placement.

Active research area (Wiebe, Granade, Jones, etc.).  SBDP slots in cleanly as
the scaling layer above their existing adaptive Bayesian-experimental-design
schemes.  Weaker baseline literature than Higgins-Wiseman: there is no single
canonical schedule analogous to geometric Ramsey.  This is double-edged: easier
to claim a methodological advance, harder to nail down a head-to-head ratio.

## Adjacent, high impact

### 4. Bayesian MRI: pulse-sequence and RF-coil co-design

T1/T2 aliasing across choices of TR, TE, and flip angle is structurally
identical to fringe aliasing.  Hardware vector:

- RF-coil geometry (100s to 1000s of dim, differentiable through Biot-Savart)
- Gradient pulse shapes
- Pulse sequence parameters (TR, TE, flip angle schedule)

Long horizon is automatic across many TRs.  Different audience entirely
(medical imaging, computational MR), large applied story (faster scans, higher
contrast, better tissue characterization).  Less explicit quantum flavor, but
the underlying math maps onto SBDP without modification.

### 5. Distributed quantum sensor networks

Joint co-design of N sensor nodes and a measurement-allocation policy for a
shared field (gravity, magnetic, or rotation).  Each SBDP batch becomes a
multi-node measurement round; the policy chooses which subset of nodes
measures, with what integration time, on each round.  The forward model is
heavier (correlations between nodes, communication latency, distributed
estimator), but no other framework handles the joint hardware-policy problem at
network scale.

## Ranking by impact-per-effort

1. **NV magnetometry** first: cheapest forward-model lift, biggest audience
   expansion, Higgins-Wiseman baseline transfers verbatim.
2. **Atom interferometry** second: new community, moderate forward-model work,
   strong commercial pull.
3. **Hamiltonian learning** third: highest novelty, weakest baseline literature
   (advance is harder to quantify).
4. **Bayesian MRI** as the dark-horse high-impact choice if leaving the
   quantum-sensing audience is acceptable.
5. **Distributed sensor networks** as a future extension paper rather than the
   immediate follow-up.

## Decision checklist for picking the next application

When evaluating any candidate beyond this list, check:

- Does the posterior multimodality persist past the exact-DP horizon?  If a
  short schedule already disambiguates, SBDP's advantage shrinks.
- Is there a canonical adaptive baseline?  Without one, the methodological
  ratio is hard to defend.
- Are the hardware DOFs differentiable end-to-end?  If forward-model
  engineering takes more than ~3 months, pick a different problem.
- Is the audience disjoint from the superconducting-qubit community?  Reusing
  the same audience halves the marginal impact of a second paper.
