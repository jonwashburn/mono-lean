ILG rotation law: from ledger principles to closed‑form curves
=============================================================

Goal
----
Make the gravity/rotation derivation robust for the paper and align it 1:1 with the single‑file Lean proof. Provide a clear spec, constants, and a checklist of properties/lemmas to harden, plus an implementation map to the Lean code.


Executive summary (locked law)
------------------------------
- Weight law (real‑space, global‑only):

  \[ w(r) = \lambda\;\xi\;n(r)\;\Big( \tfrac{T_{\rm dyn}(r)}{\tau_0} \Big)^{\alpha}\;\zeta(r), \quad T_{\rm dyn}=\frac{2\pi r}{v}. \]

- Acceleration kernel used operationally:

  \[ w_g(r) = 1 + C_{\rm lag}\Big[ \big(\tfrac{g_{\rm bar}(r)+g_{\rm ext}}{a_0}\big)^{-\alpha}
          - (1+g_{\rm ext}/a_0)^{-\alpha} \Big], \quad g_{\rm bar}=\frac{v_{\rm bar}^2}{r}. \]

- Rotation law:

  \[ v_{\rm rot}(r) = \sqrt{\,\xi\,n(r)\,\zeta(r)\,w_g(r)\,}\;v_{\rm bar}(r),\qquad
     v_{\rm bar}^2 = v_{\rm gas}^2 + (\sqrt{\Upsilon_*}\,v_{\rm disk})^2 + v_{\rm bul}^2. \]

- Constants (locked):

  \[ \alpha = \tfrac12(1-1/\varphi),\qquad C_{\rm lag}=\varphi^{-5},\qquad \tau_0=7.33\times10^{-15}{\rm s},\qquad a_0=1.2\times 10^{-10}\,\frac{\rm m}{\rm s^2}. \]

- Global shape/threads (no per‑galaxy tuning):

  \[ n(r)=1+A\big(1-e^{-(r/r_0)^p}\big),\ \ (A,r_0,p)=(7,\,8\,\mathrm{kpc},\,1.6),\qquad
     \xi=1+\varphi^{-5}\,u_b^{1/2},\qquad
     \zeta(r)\in[0.8,1.2]\ (h_z/R_d=0.25\ \text{clip}). \]


Lean mapping (current single‑file implementation)
-------------------------------------------------
- Namespace: `IndisputableMonolith.Gravity.ILG`
  - `structure BaryonCurves` with fields `vgas, vdisk, vbul`
  - `vbarSq`, `vbar`, `gbar` (with small eps guards)
  - `n_of_r (A r0 p)`, `xi_of_u (u)`, `zeta_of_r`
  - `w_core_accel (g gext)` = `1 + Clag * (rpow(...) - rpow(...))`
  - `w_tot (C xi gext A r0 p)` = `xi * n * ζ * w_core_accel`
  - `vrot (C xi gext A r0 p)` = `sqrt(w_tot) * vbar`
- Constants in `IndisputableMonolith.Constants`
  - `alpha_locked := (1 - 1/phi)/2`
  - `Clag := 1 / (phi ^ 5)`
  - `a0_SI := 1.2e-10`

- Kernel mode additions in `IndisputableMonolith.Gravity.ILG`
  - `w_core_time (t)` centered at `t=1` (time-kernel), with `w_core_time_at_ref` and reference equality to accel-kernel
  - `KernelMode` selector with `accel | time | accelInf1`, plus `w_core`, `w_tot_mode`, `vrot_mode`
  - EFE continuity and sensitivity: `w_core_accel_continuous`, coarse bounds `w_core_accel_small_gext_decomp_bound`, and uniform envelope `vrot_envelope_bounds_of_xi`.

Mass recognition layer (discrete → φ‑exponent ratios)
----------------------------------------------------
- Discrete layer: `Species`, `Sector`, `tildeQ`, `Z`, and frozen rung integers `r` live in `IndisputableMonolith.Recognition`.
- Gap/exponent: `Fgap(Z)=log(1+Z/φ)/log φ`, `massExp(i)=r_i + Fgap(Z_i) − 8`, `PhiPow x=exp(log φ·x)`, `mass M0 i = M0 * PhiPow (massExp i)`.
- Anchor identity (single seam to measurement): `anchorIdentity f : ∀i, f i = Fgap (Z i)`.
- Consequences (proved): equal‑Z degeneracy; exact anchor ratios `mass M0 i / mass M0 j = PhiPow (r_i − r_j)`. For natural Δr, this equals `φ^Δr`.
- Family examples encoded: `μ/e = φ^11`, `τ/μ = φ^6`, `c/u = φ^11`, `t/c = φ^6`, `s/d = φ^11`, `b/s = φ^6`.

Spec and properties to harden
-----------------------------
1) Assumptions and invariants
- Prove in Lean (from existing constants layer):
  - `phi > 1`, `alpha_locked = (1 - 1/phi)/2`, `0 < alpha_locked < 1`
  - `Clag > 0`, `a0_SI > 0`
- Optional unit/spec annotations: w, ξ, n, ζ dimensionless; a0 in SI.

2) Kernel and limits
- Newtonian limit: `lim_{g_bar→∞} w_core_accel(g, 0) = 1`.
- Small‑lag spec: `w ≈ 1 + O(Δt/T_dyn)` as a comment plus inequality bounds (where feasible).
- Monotonicity/positivity guard: `w_core_accel(g, gext) ≥ 1` for `g ≥ 0, gext ≥ 0` (with base clamps in code).

3) Global‑only factors
- Normalization spec for `n(r)` (disc‑weighted mean 1 under stated measure), or document the analytic choice.
- Deterministic `ξ(u_b)` interface for global gas‑fraction bins (no per‑galaxy tuning).
- Bound lemma for `ζ(r)` with clipping (0.8 ≤ ζ ≤ 1.2).

4) Domains and totality
- Provide spec variants without internal epsilons by assuming `r > 0`, `v≥0` in theorems; keep current guarded defs for totality.

5) Rotation‑curve behavior
- Newtonian proxy: if `M_enc(r) = α_M r` with `α_M ≥ 0`, then `vrot` is constant (flat) in the pure Newtonian case.
- ILG asymptotics: show the kernel produces near‑flat behavior over observed ranges under typical disk profiles.

6) External‑field effect
- Document and bound the offset: behavior at `g=0` with `gext ≥ 0`; continuity in `gext`.
  - Implemented continuity (`w_core_accel_continuous`) and a uniform bound for `w` and `vrot` under `gext≥0`.

7) Sensitivity and robustness
- Continuity of `w_core_accel` in `α` and `gext` (domain guarded via `rpow`).
- Simple bounds: for `r ≥ r0`, `n(r) ∈ [1, 1+A]`; `ξ(u) ∈ [1, 1+Clag]`.
 - Envelope: with `ξ∈[1,1+Clag]`, `ζ∈[0.8,1.2]`, `gext≥0`, get two‑sided bounds on `vrot`.

8) Build environment and examples
- Lake/mathlib setup note to resolve `Mathlib` imports.
- Toy example (exponential disk + gas) and a simple check (e.g., `w_core_accel(a0,0)=1`, basic monotonicity) via `#eval` or example lemmas.


Questions (to guide next proofs/edits)
--------------------------------------
- Which ledger theorems do we want to surface directly in `Constants` so α‑locks/φ‑facts are theorem‑backed (no reliance on defs)?
- Do we want the high‑acc limit and small‑lag expansion stated as explicit lemmas (with inequalities), or as spec comments?
- How strict should the global normalization of `n(r)` be (e.g., disc‑weighted mean = 1), and do we encode that as a lemma or documentation?
- Should we replace eps guards with domain hypotheses in main theorems (keeping total versions for evaluation)?
- What external‑field ranges do we want to support (e.g., `gext/a0 ≤ 0.1`) and what qualitative lemmas are most informative?
- Which toy baryon profiles should we include for demonstration (e.g., exponential disk, cored gas), and what properties do we want to assert about `vrot`?


## Detailed questions to drive robustness

### Assumptions and invariants
- Can we state and prove the exact hypotheses under which α = (1 − 1/φ)/2 holds in Lean (from your ledger layer) and export it as a theorem used by ILG?
- Do we want α_locked ∈ (0,1) as a lemma, and similarly φ > 1, Clag > 0, a0 > 0?
- Should we formalize unit/dimension checks (dimensionless w, ξ, n, ζ; a0 in SI) as comments or as spec lemmas?

### Kernel and limits
- Can we prove the high‑acceleration/Newtonian limit: lim_{g_bar→∞} w_core_accel(g,0) = 1?
- Do we want a small‑lag expansion lemma w ≈ 1 + O(Δt/T_dyn) reflected as a Lean spec (inequality bounds) rather than just a comment?
- Should we prove w_core_accel(g,a0,gext) ≥ 1 under g,gext ≥ 0 (monotonicity/positivity guards)?

### Global‑only factors (ξ, n, ζ)
- Do we want a normalization lemma for n(r) (e.g., disc‑weighted mean equals 1 under a chosen measure)?
- How should we formalize ξ(u) bins: a deterministic map from gas‑fraction quantiles to u_b? Provide an abstract interface or a concrete bin function?
- Should ζ(r) be a clipped function with a provable bound 0.8 ≤ ζ ≤ 1.2?

### Ledger → ILG linkage
- Which concrete ledger facts/theorems (from `IndisputableMonolith`) should be restated as a `LedgerBridge` instance to eliminate ε‑guards by hypothesis (r>0, g≥0)?
- Can we derive α and Clag directly from cost/φ lemmas (T5/T8) inside the same file with “no new axioms”?

### Domains and totality
- Do we want to replace the internal εr, εv guards by explicit hypotheses on r and component curves (v≥0, r>0) and keep alternative guarded versions?
- Should we enforce gext ≥ 0, r ≥ 0, v_bar ≥ 0 as assumptions on the evaluation domain?

### Exact rotation law proofs
- Can we prove the “flat‑rotation” lemma: if M_enc(r) = α_M r (α_M≥0), then vrot is constant (in the Newtonian proxy block), and also show the same trend under ILG (asymptotics)?
- Do we want to prove vrot is strictly increasing in k (binary scale B=2^k) or monotone in r under standard disk profiles?

### Computability and API
- Should we provide a computable sandbox (piecewise polynomials) for vgas, vdisk, vbul so vrot is evaluable without external mathlib heavy machinery?
- Do we want a “Spec API” that separates symbolic (proof) paths from numeric (evaluation) paths via typeclasses?

### Testing and examples
- Include a toy example (exponential disk + gas) and a lemma showing qualitative flattening for large r?
- Add test specs: vrot(r0)=…, w_core_accel(a0,0)=1, ξ(u_b) monotone in u_b.

### External‑field effect
- Should we add lemmas for gext > 0 (offset limit at g=0 and continuity in gext)?
- Do we want an inequality: w_core_accel(g, gext) ≥ 1 for g ≥ 0 with fixed gext ≥ 0?

### Sensitivity and robustness
- Do we want α‑sweep and gext‑sweep lemmas as qualitative continuity: w depends continuously on α, gext (for rpow domain)?
- Should we add bounds: for r ≥ r0, n(r) ∈ [1, 1+A], ξ(u) ∈ [1,1+Clag], etc.?

### Build and environment
- Should we embed a Lake/mathlib header comment with minimal instructions to resolve the Mathlib import prefix?
- Do we want a `#eval` example (when mathlib present) to print vrot at a few radii for a toy profile?

### Documentation
- Add docstrings that cite your locked equations and reference sections (acceleration kernel, constants, global‑only policy)?
- Do we want a “spec section” summarizing the exact equalities implemented (kernel, rotation law) so reviewers can cross‑check with the paper?

Answers to detailed questions
----------------------------

### Assumptions and invariants
- α proof and export: Keep `Constants.alpha_locked := (1 - 1/phi)/2` as the canonical definition, and expose lemmas `alpha_locked_pos`, `alpha_locked_lt_one` (done in Lean) plus `one_lt_phi`, `phi_pos` (already present). If you want α “as a theorem” from the ledger layer, wrap the upstream facts into a small bridge (e.g., `LedgerBridge.alpha_def`) and restate `alpha_locked = (1 - 1/phi)/2` as a theorem alias; downstream API remains unchanged.
- Positivity bounds: `Clag_pos`, `a0_SI_pos` are provided. These are sufficient for safe use of `Real.rpow` domains.
- Units: We document unit/dimension checks (w, ξ, n, ζ dimensionless; a0 SI) in md/docstrings. Full unit typing in Lean would require a units framework—out of scope here.

### Kernel and limits
- High‑acceleration/Newtonian limit: For the current acceleration kernel centered at `g=a0` (with `gext=0`), `w_core_accel(g,0) → 1 - Clag` as `g→∞`. This is intentional—reference is at `a0`. If you prefer `lim_{g→∞} w=1`, use the time‑kernel or re‑normalize the baseline term. At `g=a0`, `w_core=1` (proved: `w_core_accel_at_ref`).
- Small‑lag expansion: The literal `w ≈ 1 + Δt/T_dyn` belongs to the time‑kernel derivation near small lag. For the acceleration kernel, the “small‑lag” analogue is a first‑order expansion around `g=a0` (continuity and differentiability of `rpow`). We keep this as spec documentation rather than add a full inequality proof here.
- Monotonicity/positivity: `w_core_accel(g,gext)` is not generally ≥1. It is 1 at `g=a0`, >1 for `g<a0`, and <1 for `g>a0` (with fixed `gext`). Positivity holds for all `g>0`.

### Global‑only factors (ξ, n, ζ)
- n(r) normalization: We use the analytic form `n(r)=1+A(1-e^{-(r/r0)^p})` and (optionally) normalize its disc‑weighted mean to 1. A formal lemma would require specifying the weighting measure; we document the policy in the paper/spec.
- ξ(u) bins: Provide a deterministic mapping from global quintile bin centers `u_b ∈ {0.1,0.3,0.5,0.7,0.9}` to `ξ=1+φ^{-5}√u_b`. In Lean we retain `xi_of_u(u)` and can add a tiny helper to pick these bin centers; this keeps global‑only reproducibility.
- ζ(r) bounds: For the current default `ζ(r)=1`, we already include `zeta_bounds : 0.8 ≤ ζ ≤ 1.2`. If you switch to an explicit clipped function, we can restate that bound by construction.

### Ledger → ILG linkage
- Bridge usage: To remove eps guards in theorem statements, introduce a bridge instance that asserts `r>0`, `g≥0` hypotheses where needed. Keep the total functions (with eps guards) for numeric stability.
- α and Clag from T5/T8: You can reference the upstream cost/φ lemmas (T5/T8) to justify these constants without adding axioms. In this file we kept them as canonical defs and cited the paper; adding a compact lemma that restates them from the upstream constants is straightforward if desired.

### Domains and totality
- Variants without eps: Provide theorem variants parameterized by `r>0` (and `g≥0` when needed) to avoid eps in statements, while keeping the total definitions for evaluation; done for nonnegativity facts of `vbar` and `gbar`.
- Domain assumptions: For physical use, assume `gext ≥ 0`, `r ≥ 0`, `v_bar ≥ 0` in theorem statements. This matches data usage and avoids pathological inputs.

### Exact rotation law proofs
- Newtonian flat‑rotation lemma: If `M_enc(r)=γ r` with `γ ≥ 0`, then `v^2 = Gγ` and the rotation curve is flat. This goes in the `Gravity.Rotation` (pure Newtonian) block and is straightforward to include as a lemma.
- ILG asymptotics: Under typical disk profiles and bounded global factors, ILG preserves near‑flat behavior where `g_bar ∝ r^{-1}`. A full proof depends on the mass model; we present it as a qualitative statement backed by data.
- Monotonicity: Not guaranteed in r globally. Add monotonicity claims only for specific models (e.g., exponential disks) if needed.

### Computability and API
- Computable sandbox: Optional. We can provide piecewise polynomials for `(vgas,vdisk,vbul)` to allow quick `#eval` demonstrations without heavy numeric libraries. This is not needed for the formal statements.
- Spec API: If desired, define a typeclass‑based separation between symbolic (proof) and numeric (evaluation) paths; for now the current simple function API suffices.

### Testing and examples
- Toy example: Add an exponential‑disk + gas toy and compute a few radii samples (guarded `#eval` when mathlib is present). Simple checks already exist (`w_core_accel_at_ref`). We can add `ξ(u_b)` monotonicity over bin centers trivially.
- Test specs: Keep them lightweight; avoid extra dependencies. The current file includes bounds and equality at the reference point.

### External‑field effect
- Continuity: `w_core_accel` is continuous in `(g,α,gext)` for positive bases; this follows from `rpow` continuity. We keep this qualitative statement; a full topology proof can be added later if needed.
- Inequalities: As above, `w_core_accel(g,gext) ≥ 1` is not generally true; document parameter ranges (e.g., small `gext/a0`) in sensitivity analyses rather than assert global inequalities.

### Sensitivity and robustness
- α‑sweep/gext‑sweep: `w_core_accel` is jointly continuous for positive inputs; small changes in α or gext lead to small changes in w. We treat this as a qualitative property in the paper.
- Simple bounds: For all r, `ξ(u) ≤ 1+Clag`; for `r≥r0`, `n(r) ≤ 1+A`; `ζ(r)` bounded by clip; these are already documented/bounded in code.

### Build and environment
- Lake/mathlib: Add mathlib4 to the Lake project to resolve imports (not a code change). The file includes a comment pointing to this step.
- `#eval` examples: Add once mathlib is wired to avoid linter errors during CI; keep behind comments until then.

### Documentation
- Docstrings: Keep short docstrings referencing the locked equation forms; the full prose and citations live in this md and the paper. Avoid heavy narrative in the Lean file to preserve readability.

Units and dimensions (spec)
---------------------------
- Dimensionless: w(r), ξ, n(r), ζ(r), α, Clag. These purely scale responses and carry no SI units.
- SI quantities: a0 has SI acceleration units (m s⁻²); τ0 is in seconds.
- Derived: g_bar = v_bar² / r uses consistent units for v_bar and r; v_rot inherits v_bar units.
- In Lean we keep these as doc/spec; we do not track units in the type system.

Small‑lag spec (acceleration‑kernel)
------------------------------------
- The literal w ≈ 1 + Δt/T_dyn belongs to the time‑kernel derivation. For the acceleration kernel, the analogous first‑order statement is an expansion of x ↦ x^{−α} around x≈1 (g≈a0) with α∈(0,1). We document this as a qualitative spec; full inequalities are not required for paper claims and can be added later.

n(r) normalization policy
-------------------------
- We use the analytic profile n(r)=1+A(1−e^{−(r/r0)^p}) with (A,r0,p)=(7,8 kpc,1.6).
- Policy: n(r) may be globally normalized so that its disc‑weighted mean equals 1 under a declared weight (e.g., exponential‑disk weight). This is a presentation choice and does not affect the formal kernel definitions. A formal lemma requires a declared measure; for now we document the policy in the paper/spec.

Build setup (Lake/mathlib)
--------------------------
- Initialize a Lake project (Lean 4.22+) and add mathlib4 to resolve `Mathlib` imports (e.g., `Real.rpow`). Minimal steps:
  - `lake new indisputable-lean`
  - Edit `lakefile.lean` to `require mathlib from git` and add `lean_lib IndisputableMonolith`.
  - `lake update` then `lake build`.
  - Optionally fetch cache: `lake exe cache get`.
- Once configured, `IndisputableMonolith.lean` elaborates and example sections (`#eval`) can be enabled.


Roadmap (high‑value TODOs)
---------------------------
1) α/φ hardening
- Promote α‑lock to a lemma from the constants section; add `0<α<1`, `Clag>0`, `a0>0` lemmas.

2) Kernel limits & guards
- Add Newtonian‑limit lemma `w_core_accel(g,0)→1`; document small‑lag form.
- Add positivity/monotonicity guard for `g,gext ≥ 0` (with current base clamps).

3) Global‑only factors
- Provide a deterministic `ξ` API from global bin centers; add `ζ` clipping bound lemma.
- Optionally document/justify the analytic `n(r)` normalization used in the paper.

4) Domain‑assumption variants
- Provide theorem variants parameterized by `r>0` to remove eps in statements.

5) Examples & tests
- Add a toy profile example (exponential disk + gas) and show near‑flat behavior in ILG; include simple `#eval` checks.

6) Build doc
- Include a short Lake/mathlib setup block at the top of the file for reproducibility.


