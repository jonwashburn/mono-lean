Indisputable Monolith (Lean 4) – ILG Rotation Law and Gravity
=============================================================

This repository contains a single‑file Lean development (`IndisputableMonolith.lean`) with:
- recognition/ledger primitives and constants (φ, cost uniqueness),
- gravity scaffolding (Newtonian rotation),
- ILG rotation law skeleton (acceleration kernel w, global‑only factors),
- spec lemmas and bounds (α, C_lag, a0 positivity; ξ, n, ζ bounds),
- optional variant kernel with strict Newtonian limit at ∞.


Build (Lean 4 + mathlib4)
-------------------------
1) Create a Lake project (Lean 4.22+):

```
lake new indisputable-lean
cd indisputable-lean
```

2) Edit `lakefile.lean` to add mathlib4 and this library:

```lean
import Lake
open Lake DSL

package indisputable where

lean_lib IndisputableMonolith

require mathlib from git
  "https://github.com/leanprover-community/mathlib4" @ "master"
```

3) Bring the Lean file into `./IndisputableMonolith.lean` (or add a proper path and update `lean_lib`).

4) Fetch deps and build:

```
lake update
lake exe cache get    # optional cache for mathlib4
lake build
```

You should now be able to elaborate the file and enable optional `#eval` examples by uncommenting them.


Notes
-----
- Units: w, ξ, n, ζ, α, C_lag are dimensionless; a0 has SI acceleration units; τ0 is seconds.
- The acceleration kernel is centered at `g=a0`. If you prefer `lim_{g→∞} w = 1`, use the provided `w_core_accel_inf1` or the time‑kernel variant referenced in the paper.
- The development avoids heavy numeric code; proofs emphasize invariants, bounds, and exact equalities used by the paper.


