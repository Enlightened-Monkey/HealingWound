# WOUND HEALING SIMULATION — FULL THEORY & IMPLEMENTATION GUIDE

This document is the complete blueprint and technical reference for the project:
what each block does, how it is implemented, and which equations define behavior.

## 1. Introduction & Theory

### 1.1 Wound Healing Simulation Overview
The model is a 2D lattice process with coupled subsystems:
1. **Geometry subsystem**: builds wound shape and depth map.
2. **Healing subsystem**: walkers visit cells and heal them after threshold visits.
3. **Infection subsystem**: infection spreads, accumulates damage, and kills cells.
4. **Coupling subsystem**: healing and infection compete over the same cells.

### 1.2 Mathematical Framework (LaTeX)
**Brownian motion / random-walk foundation**:
$$
d\mathbf{X}_t = \sqrt{2D}\,d\mathbf{W}_t
$$
Discrete approximation (per time step $\Delta t$):
$$
\mathbf{X}_{t+1} = \mathbf{X}_t + \sqrt{2D\Delta t}\,\boldsymbol{\xi}_t,\quad \boldsymbol{\xi}_t \sim \mathcal{N}(0,I)
$$

**Classic random walk (CRW)** on lattice:
$$
\mathbf{x}_{t+1} = \mathbf{x}_t + \Delta\mathbf{x}_k,\quad
\Pr(k)=\frac{1}{|\mathcal{N}(\mathbf{x}_t)|}
$$

**Persistent random walk (PRW)**:
$$
\mathbf{h}_t = \alpha\mathbf{h}_{t-1} + (1-\alpha)\boldsymbol{\eta}_t,\quad
\boldsymbol{\eta}_t \sim \mathcal{N}(0,I)
$$
$$
\mathbf{x}_{t+1}=\arg\max_{\Delta\mathbf{x}_k}\left(\hat{\Delta\mathbf{x}}_k\cdot \mathbf{h}_t + \epsilon_k\right)
$$

**Healing condition**:
$$
\text{healed}(x) \Longleftrightarrow \text{visits}(x)\ge \text{threshold}(x)
$$

**Infection counter dynamics**:
$$
\text{counter}_{t+1}(x)=\text{counter}_{t}(x)+1 \quad \text{with probability } p_{\text{regrow}}
$$
$$
\text{dead}(x)\Longleftrightarrow \text{counter}(x)\ge \tau_{\text{death}}
$$

## 2. Wound Geometry

### 2.1 Theory: Wound shape generation and boundaries
The wound boundary is generated in polar form:
$$
r(\theta)=r_0+\sum_{k=1}^{K}\left[a_k\cos(k\theta)+b_k\sin(k\theta)\right]+\epsilon(\theta)
$$
where $\epsilon(\theta)$ is smoothed stochastic perturbation.

### 2.2 Step 1: Noisy boundary generation
**What it does**: generates boundary points $(x(\theta),y(\theta))$ with harmonics and noise.  
**How it works**:
1. Sample $\theta \in [0,2\pi)$.
2. Build $r(\theta)$ from harmonic + noise term.
3. Convert polar $\to$ Cartesian points.

### 2.3 Step 2: Rasterization to binary mask
**What it does**: converts contour points into discrete wound region.  
**How it works**:
1. Draw line segments between consecutive contour points.
2. Fill interior region.
3. Result: binary mask `wound_mask`.

### 2.4 Step 3: Boundary ring extraction
**What it does**: extracts wound edge cells.  
**How it works**:
$$
\text{boundary\_ring} = \text{mask} \land \neg\text{erode}(\text{mask})
$$
This ring defines boundary-adjacent cells.

### 2.5 Step 4: BFS distance map
**What it does**: computes depth of each wound cell from boundary.  
**How it works**:
1. Use BFS/EDT from boundary.
2. Assign distance $d(x)$ to each wound cell.
3. Store in `dist_to_boundary`.

### 2.6 Step 4b: Wound texture (Perlin noise)
**What it does**: modulates depth map with 2D Perlin FBM.  
**Perlin composition**:
$$
\text{Perlin}_{\text{fbm}}(x,y)=\sum_{o=0}^{O-1} a_o\,n(2^o x,2^o y),\quad a_o=2^{-o}
$$
**Modulation**:
$$
\text{depth}_{\text{textured}}(x)=\text{depth}_{\text{base}}(x)\cdot\left[1+s(2P(x)-1)\right]
$$
where $P(x)\in[0,1]$ is Perlin map and $s$ is texture strength.

## 3. Random Walk Framework & Healing Mechanism

### 3.0 Theory: Healing Counter & Depth-Dependent Thresholds
$$
\text{threshold}(d)=\text{min\_thr}+(\text{max\_thr}-\text{min\_thr})\cdot\frac{d}{d_{\max}}
$$
Interpretation:
1. Near boundary: low threshold (easy healing).
2. Deep center: high threshold (hard healing).

### 3.1 Theory: CRW vs PRW with LaTeX formulas
**CRW**:
$$
\Pr(\Delta \mathbf{x}_k)=\frac{1}{N_{\text{valid}}}
$$
**PRW**:
$$
\mathbf{h}_t=\alpha\mathbf{h}_{t-1}+(1-\alpha)\boldsymbol{\eta}_t
$$
**Comparison metrics**:
$$
\text{displacement}=\|\mathbf{x}_T-\mathbf{x}_0\|
$$
$$
\text{tortuosity}=\frac{\sum_{t=1}^{T}\|\mathbf{x}_t-\mathbf{x}_{t-1}\|}{\|\mathbf{x}_T-\mathbf{x}_0\|+\varepsilon}
$$

### 3.2 Demo 1: Path comparison (CRW vs PRW)
**Output**: static side-by-side trajectories + displacement/tortuosity.

### 3.3 Demo 2: Drift sector visualization
**Output**: sector-box context for current walker position and heading direction.

### 3.4 Step 5: Healer agent spawn points
**What it does**: chooses distributed start points on boundary ring (k-means style).  
**How it works**:
1. Cluster boundary points into $k$ groups.
2. Use centroids as spawn seeds.

### 3.5 Depth-dependent healing thresholds
**Output**: threshold map visualization (boundary→deep gradient).  
Each cell stores required visit count before it can heal.

### 3.6 Simulation and visualization
`run_nodrift_healing(...)` executes healing-only baseline:
1. Move walkers in wound mask.
2. Update visit counters.
3. Flip healed state when threshold reached.
4. Save snapshots for visualization/video.

## 5. Infection Dynamics

### 5.1 Theory: Infection spread, counter system, death mechanics
States: healthy, infected, dead.  
Spread rule:
$$
\Pr(\text{infect }x \mid \exists\,\text{infected neighbor})=p_{\text{spread}}
$$
Counter growth and death:
$$
\text{counter}_{t+1}=\text{counter}_{t}+1\quad\text{with prob }p_{\text{regrow}},\qquad
\text{dead}\iff \text{counter}\ge\tau_{\text{death}}
$$
Infected/dead cells block or reverse healing.

### 5.2 Step 6: Infection point selection
Seeds are sampled with depth weighting:
$$
w(x)\propto \left(\frac{d(x)}{d_{\max}}\right)^\beta
$$
Higher-depth cells can be favored.

### 5.3 Step 6b: Infection-only progression
`run_infection_only(...)` simulates spread/death without walkers:
1. Initialize seeds.
2. Propagate by neighborhood probability.
3. Increase counters and kill at threshold.
4. Save snapshots.

### 5.4 Step 6c: Sector grid for infection scoring
Grid partition:
$$
\text{sector}(r,c)=\left(\left\lfloor\frac{r}{s}\right\rfloor,\left\lfloor\frac{c}{s}\right\rfloor\right)
$$
Per-sector infection density:
$$
\rho_{\text{inf}}(S)=\frac{|\{x\in S:\text{infected}(x)\}|}{|S|}
$$

## 6. Walker Death & Cell Isolation Mechanics

### 6.1 Theory: Walker trapping and gradual cell necrosis
**Trapped walker removal**:
$$
|\mathcal{N}_8(\mathbf{x}_t)\cap \text{active\_mask}|=0 \Rightarrow \text{walker removed}
$$
**Isolation necrosis**:
$$
\text{isolation\_counter}_{t+1}(x)=\text{isolation\_counter}_t(x)+1\quad\text{if }x\text{ disconnected from boundary}
$$
$$
\text{cell dies}\iff \text{isolation\_counter}(x)\ge\tau_{\text{iso}}
$$

### 6.2 Helper functions
1. `compute_boundary_reachability(...)`: connected-component reachability from grid edge.
2. `apply_isolation_necrosis(...)`: counter update + death transition.
3. `remove_trapped_walkers(...)`: drops walkers with no valid neighbors.

### 6.3 Integration into main loop
Order in each step:
1. Walker move/heal.
2. Trapped walker filtering.
3. Infection update.
4. Reachability + isolation necrosis update every $N$ steps.

## 7. Main Simulation: Healing + Infection Dynamics

### 7.1 Theory: Persistent RW with drift, sector scoring, neighbor healing
Sector priority:
$$
\text{score}_{\text{sector}}=w_{\text{inf}}\cdot \rho_{\text{inf}} + w_{\text{depth}}\cdot \bar d
$$
PRW motion:
$$
\mathbf{h}_t=\alpha\mathbf{h}_{t-1}+(1-\alpha)\boldsymbol{\eta}_t,\quad
\mathbf{x}_{t+1}=\arg\max_k(\hat{\Delta\mathbf{x}}_k\cdot\mathbf{h}_t+\epsilon_k)
$$
Coupled state update includes healing, infection spread, death, walker removal, and isolation.

### 7.2 Implementation: `run_walker_healing_with_infection()`
State tensors:
1. `visits`, `healed`
2. `infected`, `dead`, `counter`
3. `walkers`, `heading`
4. `isolation_counter`, `threshold_map`

Function returns snapshots + final arrays + walker-death log.

### 7.3 Demo 3: Highlighted walker in wound
Visualization emphasizes one walker trajectory, local sector context, and live metric readout.

### 7.4 Full simulation visualization
Color semantics (final maps/frames):
1. Healed → green
2. Active infection → yellow/orange
3. Dead → dark/gray
4. Base wound → red scale
5. Walkers → cyan/blue

## 8. Comparison & Results

### 8.1 Metrics & Evaluation
Core metrics:
$$
\text{healing\_efficiency}=\frac{N_{\text{healed}}}{T}
$$
$$
\text{infection\_resistance}=\frac{N_{\text{healed and not dead}}}{N_{\text{healed}}+\varepsilon}
$$
Additional: visit histogram, spatial coverage, walker survival.

### 8.2 Comparative Analysis
Compare CRW vs PRW vs no-drift by:
1. displacement/tortuosity
2. healing curves over time
3. infection/death trajectories
4. isolation-death event timeline

## 9. Long-form Video Export (MP4)

### 9.1 MP4 rendering pipeline
1. Convert each snapshot into RGB frame.
2. Encode ordered frames with fixed FPS.
3. Save as long-form MP4 artifact.

Duration:
$$
\text{duration [s]}=\frac{N_{\text{frames}}}{\text{fps}}
$$

This fully replaces interactive viewer needs for presentation/review scenarios.
