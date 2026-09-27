# Quantum-Enhanced Adaptive Urban Traffic Optimization 🚦⚛️

An experimental traffic-signal optimization platform that combines **urban traffic simulation, QUBO modeling, Ising Hamiltonians, and QAOA** to investigate adaptive signal control under changing traffic conditions.

The system is designed as a modular research platform rather than a claim of practical quantum advantage. It supports both a **classical rule-based controller** and a **QAOA-based optimizer**, allowing their behavior and computational characteristics to be evaluated under the same simulated traffic scenarios.

---

## Overview

Urban traffic signals are commonly controlled using fixed or manually tuned timing plans. Such strategies can struggle when traffic demand changes rapidly, congestion propagates between intersections, or emergency vehicles require priority.

This project models traffic signal selection as a constrained optimization problem.

For every controlled intersection, the optimizer selects:

* Traffic phase: **North–South** or **East–West**
* Green duration: **30, 60, or 90 seconds**

These decisions are encoded as binary variables and assembled into a **QUBO (Quadratic Unconstrained Binary Optimization)** problem.

The QUBO is then transformed into an **Ising Hamiltonian** and passed to a **QAOA circuit implemented with Qiskit Aer**.

The resulting system can be evaluated against a classical rule-based controller using metrics such as:

* Average waiting time
* Queue length
* Throughput
* Fuel consumption
* CO₂ emissions
* Emergency-vehicle travel time
* Optimization runtime
* QAOA feasibility and fallback behavior

---

## System Architecture

```text
                    ┌──────────────────────────┐
                    │      Traffic State       │
                    │ queues / density /       │
                    │ phase / emergency data   │
                    └────────────┬─────────────┘
                                 │
                  ┌──────────────▼──────────────┐
                  │     Traffic Controller      │
                  └──────────────┬──────────────┘
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
      ┌───────▼────────┐                   ┌────────▼────────┐
      │ Classical      │                   │ Quantum         │
      │ Rule-Based     │                   │ QAOA Controller │
      │ Controller     │                   │                 │
      └───────┬────────┘                   └────────┬────────┘
              │                                     │
              │                            ┌────────▼────────┐
              │                            │ Traffic Cost     │
              │                            │ Model            │
              │                            └────────┬────────┘
              │                                     │
              │                            ┌────────▼────────┐
              │                            │ QUBO Builder     │
              │                            └────────┬────────┘
              │                                     │
              │                            ┌────────▼────────┐
              │                            │ QUBO → Ising     │
              │                            └────────┬────────┘
              │                                     │
              │                            ┌────────▼────────┐
              │                            │ QAOA / Qiskit    │
              │                            │ Aer Simulator    │
              │                            └────────┬────────┘
              │                                     │
              └──────────────────┬──────────────────┘
                                 │
                         ┌───────▼────────┐
                         │ Traffic        │
                         │ Simulation     │
                         └───────┬────────┘
                                 │
                 ┌───────────────▼────────────────┐
                 │ SUMO / TraCI + internal        │
                 │ simulation infrastructure      │
                 └────────────────────────────────┘
```

---

## Core Optimization Formulation

For each intersection \(i\), phase \(p\), and duration \(d\), the system defines a binary variable:

$$
x_{i,p,d} \in \{0,1\}
$$

where:

* \(p \in \{NS, EW\}\)
* \(d \in \{30,60,90\}\)

This gives **6 candidate configurations per intersection**.

For \(N\) optimized intersections:

$$
6N
$$

binary optimization variables are generated.

For example:

| Intersections | Binary variables |
| ------------: | ---------------: |
|             1 |                6 |
|             2 |               12 |
|             3 |               18 |
|             4 |               24 |
|             8 |               48 |

### Exactly-one constraint

Each intersection must select exactly one phase-duration configuration:

$$
\sum_k x_{i,k}=1
$$

The implementation converts this into a QUBO penalty:

$$
\lambda\left(1-\sum_k x_{i,k}\right)^2
$$

so invalid multi-selection and zero-selection states receive a large penalty.

---

## Traffic Objective

The cost model is designed to capture multiple traffic objectives rather than queue length alone.

The current formulation includes components for:

### Queue pressure

Penalizes vehicles waiting on approaches affected by a candidate signal decision.

### Waiting / delay

Penalizes accumulated waiting time.

### Congestion

Uses normalized traffic density / queue pressure to discourage highly congested states.

### Throughput reward

Rewards candidate configurations that discharge more vehicles during the selected green interval.

### Downstream congestion

Penalizes releasing traffic toward heavily congested downstream intersections.

### Network coordination

Neighboring intersections receive quadratic coupling terms encouraging compatible signal phases and green-wave behavior.

### Phase switching stability

A switching penalty discourages unnecessary changes from the current signal phase.

### Emergency priority

Emergency approaches receive additional priority, while candidate configurations that keep an emergency approach red receive penalties.

### Emissions proxy

The optimization objective includes an emissions proxy based on traffic density, idling, and residual queues.

Actual fuel and CO₂ values are measured separately through the SUMO simulation layer.

---

## QUBO → Ising → QAOA

The QUBO is represented as:

$$
C(x)=x^TQx+c
$$

The system then performs the standard binary-to-spin transformation:

$$
x_i = \frac{1-Z_i}{2}
$$

which produces an Ising Hamiltonian:

$$
H(Z)=
\sum_i h_iZ_i+
\sum_{i<j}J_{ij}Z_iZ_j+
C
$$

The `IsingConverter` implementation preserves the objective equivalence between the QUBO and Ising representations for binary states.

### QAOA pipeline

```text
Traffic State
     ↓
Cost Model
     ↓
QUBO Matrix
     ↓
Ising Hamiltonian
     ↓
Initial |+⟩ state
     ↓
QAOA p-layer circuit
     ↓
Qiskit Aer simulation
     ↓
COBYLA parameter optimization
     ↓
Measurement sampling
     ↓
Feasibility checking
     ↓
Best valid signal configuration
```

The current QAOA implementation supports configurable:

* Ansatz depth \(p\)
* Measurement shots
* Classical optimizer
* Maximum optimization iterations
* Random seed

The default research configuration used in the benchmark is:

```text
p = 1
shots = 512
optimizer = COBYLA
maxiter = 15
```

---

## Feasibility and Fallback

QAOA measurement samples are not automatically guaranteed to satisfy the exactly-one constraint.

The solver therefore:

1. Evaluates sampled bitstrings.
2. Checks one-hot feasibility for every intersection.
3. Selects the lowest-cost feasible candidate.
4. Falls back to a deterministic classical candidate generator if no feasible sampled bitstring is observed.

This behavior is recorded in the returned `QAOAResult`.

Relevant telemetry includes:

* Selected bitstring
* QUBO cost
* Feasibility
* QAOA parameters
* Backend
* Execution time
* Sampled bitstring distribution
* Whether fallback was triggered
* Fallback reason

---

## Simulation Environments

The repository contains two complementary simulation approaches.

### 1. Internal deterministic simulator

The `traffic_optimizer/simulation` package provides a lightweight traffic model for development and testing.

It models:

* Vehicle arrivals
* Queue accumulation
* Queue discharge
* Signal phases
* Green durations
* Reproducible seeded simulation

This allows the optimization engine to be tested without requiring SUMO.

### 2. SUMO + TraCI

The project also integrates with **Eclipse SUMO** for microscopic traffic simulation.

The integration layer supports:

* Traffic-light control
* Vehicle tracking
* Queue and waiting-time measurements
* Throughput measurement
* Fuel and CO₂ telemetry
* Emergency-vehicle injection
* Congestion / incident events
* Simulation orchestration

The bundled SUMO network contains a coordinated **2×2 intersection grid** using intersections:

```text
I1 ─── I2
│      │
│      │
I3 ─── I4
```

The repository also contains an OSM-derived scenario under:

```text
sumo/osm/
```

including a Manhattan-based road network and generated SUMO configuration files.

---

## Controllers

### Classical Rule-Based Controller

The classical controller provides a non-quantum baseline based on traffic demand and urgency heuristics.

It is intentionally kept separate from the quantum optimizer so the two approaches can be benchmarked under identical simulation conditions.

### Quantum QAOA Controller

The quantum controller:

1. Reads the current network state.
2. Builds a traffic-aware QUBO.
3. Converts the QUBO into an Ising Hamiltonian.
4. Runs QAOA.
5. Filters infeasible measurement results.
6. Converts the selected bitstring into signal decisions.
7. Applies those decisions to the traffic simulation.

For larger networks, optimization can be partitioned into smaller intersection groups to control the qubit footprint of classical QAOA simulation.

---

## Repository Structure

```text
.
├── dashboard.py
├── main.py
├── quantum_demo.py
├── run_benchmark.py
├── run_hackathon_demo.py
├── run_task14b_benchmark.py
│
├── benchmark_results.csv
├── benchmark_results.json
├── demo_results.json
│
├── pareto_sweep.py
├── pareto.pkl
├── pareto_chart.png
│
├── integration_audit.md
├── simulation_performance_report.md
├── qaoa_explanation.md
├── qubo_explanation.md
├── osm_integration.md
├── real_data_inventory.md
│
├── sumo/
│   ├── network.nod.xml
│   ├── network.edg.xml
│   ├── network.net.xml
│   ├── routes.rou.xml
│   ├── simulation.sumocfg
│   └── osm/
│       ├── build_osm_scenario.py
│       ├── manhattan.osm
│       ├── manhattan.net.xml
│       ├── manhattan.rou.xml
│       └── simulation_osm.sumocfg
│
└── traffic_optimizer/
    ├── config.py
    │
    ├── controllers/
    │   ├── signal_controller.py
    │   └── quantum_controller.py
    │
    ├── integrations/
    │   ├── sumo_adapter.py
    │   └── orchestrator.py
    │
    ├── models/
    │   ├── intersection.py
    │   ├── road.py
    │   └── traffic_state.py
    │
    ├── network/
    │   └── traffic_network.py
    │
    ├── optimization/
    │   ├── cost_model.py
    │   └── qubo.py
    │
    ├── quantum/
    │   ├── ising_converter.py
    │   ├── qaoa_solver.py
    │   └── result.py
    │
    ├── simulation/
    │   └── traffic_simulator.py
    │
    └── tests/
        ├── test_intersection.py
        ├── test_network.py
        ├── test_qaoa.py
        ├── test_qubo.py
        ├── test_road.py
        ├── test_signal_controller.py
        ├── test_sumo_integration.py
        ├── test_traffic_simulator.py
        └── test_traffic_state.py
```

---

## Installation

### Requirements

The project uses Python and the following major ecosystem components:

* Python
* NumPy
* SciPy
* Qiskit
* Qiskit Aer
* SUMO
* TraCI
* Streamlit
* Pytest

Create a virtual environment:

```bash
python -m venv .venv
```

Activate it.

### Windows

```powershell
.venv\Scripts\activate
```

### Linux / macOS

```bash
source .venv/bin/activate
```

Install the required Python packages according to the dependency configuration used for your local environment.

SUMO must also be installed separately and its executable directory made available to the system `PATH`.

---

## Running the Project

### Classical simulator demo

```bash
python main.py
```

This launches the lightweight internal traffic simulation and demonstrates the classical controller.

---

### Quantum demonstration

```bash
python quantum_demo.py
```

This demonstrates the QUBO → Ising → QAOA workflow.

---

### Full SUMO benchmark

```bash
python run_hackathon_demo.py
```

The default benchmark compares:

```text
Classical Rule-Based
        vs
Quantum QAOA
```

under dynamic traffic events.

Useful options include:

```bash
python run_hackathon_demo.py \
    --steps 100 \
    --interval 30 \
    --shots 512 \
    --maxiter 15 \
    --quantum-intersections I1,I2
```

On Windows PowerShell, the same arguments can be supplied on one line.

---

### SUMO GUI

To visualize the simulation:

```bash
python run_hackathon_demo.py --gui
```

---

### Streamlit dashboard

```bash
streamlit run dashboard.py
```

The dashboard provides visual inspection of:

* Benchmark metrics
* Traffic network state
* QUBO structure
* Ising Hamiltonian information
* QAOA parameters
* Measurement distributions
* Performance trajectories
* Classical vs quantum comparison

---

## Testing

Run the complete test suite with:

```bash
pytest
```

SUMO-dependent tests are marked separately:

```text
sumo
```

The suite covers areas including:

* Intersection behavior
* Road state
* Network construction
* Traffic-state snapshots
* Classical controller behavior
* Traffic simulation
* QUBO generation
* QAOA execution
* SUMO/TraCI integration

---

## Benchmarking Methodology

The repository contains benchmark data and analysis artifacts produced from repeated SUMO simulations.

The documented evaluation uses:

* A 2×2 coordinated intersection network
* Multiple traffic scenarios
* Multiple random seeds
* Fixed simulation duration
* Periodic optimization
* Classical and QAOA controllers evaluated under the same simulated conditions

The recorded metrics include:

| Category      | Metrics                                    |
| ------------- | ------------------------------------------ |
| Traffic       | Waiting time, queue length, throughput     |
| Environmental | Fuel consumption, CO₂ emissions            |
| Emergency     | Emergency travel time, corridor completion |
| Quantum       | QAOA calls, QAOA runtime, qubit count      |
| Reliability   | Feasibility and fallback rate              |
| Runtime       | End-to-end wall-clock simulation time      |

Detailed experimental results are available in:

```text
simulation_performance_report.md
benchmark_results.csv
benchmark_results.json
```

---

## Current Experimental Results

The repository's documented benchmark does **not** demonstrate an operational quantum advantage on the tested CPU-based QAOA simulation.

Across the documented 45 SUMO benchmark runs, the classical controller showed lower average waiting time and queue length, higher throughput, and lower computational overhead than the QAOA configuration used in the experiment.

The QAOA implementation nevertheless demonstrates several important engineering components:

* End-to-end QUBO construction
* Constraint encoding
* Exact QUBO → Ising conversion
* Parameterized QAOA circuits
* Classical parameter optimization
* Feasibility checking
* Automatic fallback behavior
* Dynamic integration with traffic simulation
* Network-level coupling between intersections

This distinction is intentional: the project evaluates where a quantum optimization formulation works, where classical simulation becomes computationally expensive, and what engineering constraints appear before execution on real quantum hardware.

---

## Scaling Considerations

Each intersection contributes six binary variables.

Therefore:

$$
N_{\text{qubits}} = 6N_{\text{intersections}}
$$

A direct QAOA simulation for a larger network quickly becomes expensive when executed with classical statevector simulation because the state representation scales exponentially with the number of qubits.

For this reason, the benchmark implementation partitions the network into smaller optimization groups.

The documented experiments use groups of up to **3 intersections / 18 qubits** for QAOA simulation.

The network-generation code itself is capable of constructing larger abstract grids, making it possible to investigate how the optimization formulation behaves as the network grows independently of the bundled SUMO scenario.

---

## Research Questions

This repository is intended to support experimentation around questions such as:

1. Can traffic-signal timing be expressed effectively as a constrained QUBO?
2. How should downstream congestion and neighboring intersections be coupled?
3. How do emergency priorities alter the optimization landscape?
4. How large can the QAOA problem become before classical simulation becomes impractical?
5. How does QAOA behave when one-hot constraints are enforced only through penalties?
6. What trade-offs occur between queue reduction, throughput, emissions, and signal stability?
7. What changes are required before the formulation can be meaningfully evaluated on real quantum hardware?

---

## Limitations

Current limitations include:

* QAOA experiments are primarily performed using classical Qiskit Aer simulation.
* Classical statevector simulation limits practical qubit scaling.
* QAOA sampling can produce infeasible bitstrings, requiring fallback logic.
* Cost-function weights require calibration across traffic distributions.
* The bundled SUMO network is substantially smaller than a real metropolitan traffic system.
* Results are simulation-dependent and should not be interpreted as real-world traffic-system performance.
* The emissions component in the optimization objective is a proxy; measured SUMO fuel and CO₂ remain separate evaluation metrics.
* The current experimental configuration does not establish quantum computational advantage.

---

## Project Philosophy

The goal is not to label an optimizer “quantum” and assume it is better.

The project treats the quantum formulation as an engineering and research problem:

```text
Real traffic problem
        ↓
Mathematical formulation
        ↓
Constrained QUBO
        ↓
Ising Hamiltonian
        ↓
QAOA
        ↓
Simulation
        ↓
Benchmark
        ↓
Failure analysis
        ↓
Iteration
```

This makes the repository useful for studying the complete pipeline from **traffic modeling to quantum optimization and empirical validation**.

---

## Documentation

Additional technical documentation:

* [`qaoa_explanation.md`](qaoa_explanation.md) — QAOA formulation and execution details
* [`qubo_explanation.md`](qubo_explanation.md) — QUBO formulation
* [`simulation_performance_report.md`](simulation_performance_report.md) — benchmark methodology and results
* [`integration_audit.md`](integration_audit.md) — integration validation
* [`osm_integration.md`](osm_integration.md) — OSM/SUMO scenario integration
* [`real_data_inventory.md`](real_data_inventory.md) — available traffic-data artifacts

---

## License

Add the project's intended license here before publishing the repository for external use.

---

## Acknowledgements

This project builds on:

* **Qiskit / Qiskit Aer** for quantum-circuit construction and simulation
* **Eclipse SUMO** for microscopic traffic simulation
* **TraCI** for programmatic SUMO control
* **SciPy** for classical parameter optimization
* **Streamlit** for interactive visualization

---

## Status

**Experimental / Research Prototype**

The platform is suitable for experimentation, benchmarking, hackathon demonstrations, and continued research into hybrid classical–quantum traffic optimization.
