readme = """# lindblad-noise-sim

A from-scratch simulation of open quantum system dynamics using the Lindblad
master equation. Built as part of my MSc in Quantum Information Technology
to develop hands-on intuition for qubit decoherence — the central obstacle
in building useful quantum computers.

Everything runs in Python with QuTiP. No quantum hardware access needed.
The final part plugs in real calibration data from IBM's ibm_brisbane QPU
to simulate what actually happens to a qubit on a real machine.

---

## What This Is About

Real qubits don't live in isolation. They interact with their environment
constantly — losing energy to it, losing phase coherence, becoming mixed
states. This process is called decoherence and it is the reason quantum
computers make errors.

The standard mathematical framework for this is the Lindblad master equation:

    dρ/dt = -i[H, ρ] + Σ_k ( L_k ρ L_k† - ½{L_k†L_k, ρ} )

where ρ is the density matrix, H is the system Hamiltonian, and the L_k
are collapse operators that encode how the system interacts with its
environment. Two physical processes are modelled here:

- Amplitude damping (L₁ = √γ₁ · σ₋): the qubit loses energy and
  relaxes to the ground state. Characterised by T1 = 1/γ₁.

- Pure dephasing (L₂ = √γ_φ · σ_z): the qubit loses phase coherence
  without changing its energy populations. Characterised by T2*.

The total coherence decay rate is γ_total = γ₁/2 + 2γ_φ, which gives
T2* = 1/γ_total. A fundamental constraint is T2* ≤ 2T1 — dephasing can
only make things worse, never better.

---

## Project Structure

The project is split into 5 parts, each building on the previous one.

### Part 1 — Single Qubit Lindblad Simulation
The starting point. A qubit initialised in the excited state |0⟩ and
the superposition |+⟩ is evolved under amplitude damping and dephasing.
The expectation values ⟨σ_x⟩, ⟨σ_y⟩, ⟨σ_z⟩ are tracked over time,
along with the state purity Tr(ρ²).

The purity result is non-obvious: starting from a pure state, the system
passes through maximum mixedness (purity = 0.5) during the transition,
then recovers back to purity = 1.0 at the ground state. Two pure states
connected by decoherence, with maximum uncertainty in between.

### Part 2 — Analytical Validation and Bloch Sphere
The simulation is validated against exact analytical solutions of the
Lindblad equation. For the |+⟩ initial state:

    ⟨σ_z⟩(t) = -1 + exp(-γ₁ t)
    ⟨σ_x⟩(t) = cos(ω₀t) · exp(-γ_total t)
    ⟨σ_y⟩(t) = sin(ω₀t) · exp(-γ_total t)

One thing worth noting: when you define L₂ = √γ_φ · σ_z in QuTiP, the
dephasing contribution to γ_total is 2γ_φ not γ_φ. This factor of 2 comes
from the dissipator structure (σ_z² = I) and is a common source of
confusion in the literature. The residual error plot shows this clearly —
⟨σ_z⟩ matches the analytic formula to machine precision (~1e-10), while
⟨σ_x,y⟩ only match after applying the corrected γ_total.

The Bloch sphere plot shows the full trajectory: a spiral starting at the
equator (|+⟩), rotating around the z-axis from Hamiltonian precession,
shrinking inward from dephasing, and drifting down to the south pole (|1⟩,
ground state) from amplitude damping. The two noise channels act
simultaneously and independently — that is exactly what the spiral encodes.

### Part 3 — Parameter Sweeps
Systematic noise characterisation across a range of γ₁ and γ_φ values.

- γ₁ sweep (0.01 → 0.5): T1 decay curves at 6 different rates.
  Exponential fitting with scipy recovers T1 = 1/γ₁ with 0.00% error
  across all values — validating that the simulation can be used as a
  digital twin for real T1 measurement experiments.

- γ_φ sweep (0 → 0.3): Dephasing only affects coherences, not
  populations. Even a small γ_φ = 0.01 nearly halves T2* compared to
  the γ_φ = 0 case. Dephasing is more destructive than it looks.

- 2D heatmap: Both rates swept simultaneously across a 40×40 grid.
  The T2*/T1 ratio is computed at every point — it never exceeds 2.0,
  with the maximum achieved exactly at γ_φ = 0. The T2 ≤ 2T1 bound is
  proven numerically across 1600 parameter combinations.

### Part 4 — Two-Qubit Entanglement Decay
The system is extended to two qubits prepared in the Bell state:

    |Φ+⟩ = (|00⟩ + |11⟩) / √2

Both qubits are subjected to independent local Lindblad noise (no
correlated environment). Entanglement is tracked using concurrence C(t),
which runs from 1.0 (maximally entangled) to 0 (separable).

The main result is **Entanglement Sudden Death (ESD)**: the concurrence
hits exactly zero at a finite time, not asymptotically. This is distinct
from single-qubit decoherence, which always decays exponentially toward
zero but never truly reaches it in finite time. The Bell state loses its
entanglement completely while the individual qubits still retain partial
coherence.

The most striking finding comes from comparing noise types at equal T2* rates:
pure amplitude damping causes ESD, while pure dephasing at the same total
decoherence rate leads only to asymptotic decay. Same T2*, fundamentally
different fate for entanglement. The physical reason: amplitude damping
drains the |11⟩ population directly, destroying the superposition that
makes the Bell state entangled. Dephasing can only rotate phases — it
cannot change populations, so it cannot fully kill entanglement in finite time.

### Part 5 — Real IBM Hardware Data
T1 and T2* values are taken directly from the ibm_brisbane calibration table
(127-qubit Eagle QPU). No API or account needed — the calibration page is
publicly visible at quantum.ibm.com.

Typical values for ibm_brisbane qubits:
- T1 range: 156 – 231 μs
- T2* range: 87 – 143 μs
- T2*/T1 ratio: 0.56 – 0.69 (well below the 2.0 bound — dephasing-dominated)

The Lindblad rates are computed as:
    γ₁   = 1 / T1
    γ_φ  = ( 1/T2* - 1/(2·T1) ) / 2

These are plugged into the same simulation framework from Parts 1–4,
but now with the time axis in real microseconds. The resulting decay curves
show what happens on an actual IBM QPU — the qubit coherence window you
have to run a circuit before the state becomes useless.

---

## Key Findings

1. The factor-of-2 trap: Defining L = √γ_φ · σ_z gives off-diagonal
   decay at rate 2γ_φ, not γ_φ. This is physically correct but easy to
   miss — using the wrong formula gives ~50% error in T2* predictions.

2. Dephasing is more destructive than amplitude damping at equal T2* rates:
   In the parameter sweeps, adding even a small amount of γ_φ collapses
   T2* much faster than equivalent γ₁. On real IBM hardware, T2*/T1 ≈ 0.6
   across all measured qubits — meaning pure dephasing contributes more
   to decoherence than energy relaxation on current superconducting devices.

3. Entanglement Sudden Death is not the same as decoherence:
   The Bell state loses all entanglement at a finite time even though
   individual qubits are still partially coherent. Amplitude damping causes
   this; pure dephasing at the same T2* rate does not. This distinction
   matters directly for quantum error correction — two-qubit gate fidelity
   is limited by ESD, not just by single-qubit T2*.

---

## Requirements

