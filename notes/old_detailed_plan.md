# Fast-reactor pin cell in Serpent: project plan

*Written 2026-11-11 from the files in this folder: the cluster manual (`cluster.pdf`), the course slides (`elements_of_RF.pdf`, `MC_lectures.pdf`), the writing guide (`Stylistic_Principles.pdf`) and the earlier geometry plots. The cluster account itself has not been inspected yet; Phase 0 does that.*

**Conventions in this document**
- **[verify]** marks a best-estimate number or syntax detail. Check it against the Serpent manual or the cited benchmark before relying on it.
- Each phase ends with a **"Done when"** checklist and a **ready-to-paste prompt** for Claude Code.
- Formulas are written in LaTeX ($\dots$).

---

## 0. How to use this plan

| Who | What to read | Time |
|---|---|---|
| Everyone in the group | Sections 1–3 (physics, cluster, workflow) | about 30 min |
| The person driving Claude Code | Sections 2–7, plus `CLAUDE.md` | about 1 h |
| Report writers | Sections 1, 5, 9 | about 20 min |

To start working, open a terminal **on your laptop** (not on the cluster):

```bash
cd ~/Desktop/KTH/ModFysLab/Projektet
claude
```

Then type: *"Read PLAN.md. Do Phase 0."* Claude Code reads `CLAUDE.md` automatically. That file holds the cluster rules it must never break.

---

## 1. What we are building, in plain words

### 1.1 "Fuel cell" means a *fuel pin cell*

In reactor physics, a **pin cell** is the smallest repeating piece of a reactor core. It is one fuel rod (fuel pellet → small gap → cladding tube) plus the coolant that surrounds it. The earlier plot `pincell_fresh_geom1.png` is exactly that, for a water-cooled pin: green fuel, black gap, grey cladding and pink water.

The outer edge of the cell gets a **reflective boundary condition**: every neutron that leaves the cell is mirrored back in. This is mathematically the same as an *infinite* lattice of identical pins. So the model is tiny (fast to run), but it still captures the neutron physics of the fuel, cladding and coolant.

> Write "fuel pin cell" or "pin cell" in the report. "Fuel cell" usually means an electrochemical device (hydrogen fuel cell), and the examiner will notice.

### 1.2 What "k" is

From the course slides, the **effective multiplication factor** $k_{\mathrm{eff}}$ is the ratio between the neutron populations of two successive generations:

$$k_{\mathrm{eff}} = \frac{\text{neutrons in generation } n+1}{\text{neutrons in generation } n}.$$

- $k_{\mathrm{eff}} = 1$ means critical (steady state).
- $k_{\mathrm{eff}} < 1$ means subcritical (the population dies out).
- $k_{\mathrm{eff}} > 1$ means supercritical (the population grows).

Because our pin cell is infinite (reflective boundaries), no neutrons leak out. What Serpent computes is therefore the **infinite multiplication factor** $k_\infty$. It is always larger than $k_{\mathrm{eff}}$ of a real, finite core. This matters a lot for fast reactors, where leakage is large (see Section 5.4).

### 1.3 Thermal ("slow") versus fast reactors

| | Thermal reactor (PWR/BWR) | Fast reactor (our task) |
|---|---|---|
| Energy of neutrons causing most fissions | about 0.025 eV ("thermal") | about 0.1–1 MeV ("fast") |
| Moderator | Water slows neutrons down on purpose | **No moderator**; avoid light nuclei |
| Coolant | Water | Liquid sodium, lead, lead–bismuth or helium |
| Lattice | Square, wide spacing | Hexagonal (triangular pitch), **tight** spacing |
| Fuel | UO₂, about 4–5 % ²³⁵U | MOX (U,Pu)O₂ with about 15–25 % Pu, or metal, nitride or carbide |
| Why | Fission cross sections are huge at low energy | Pu-239 releases more neutrons per absorbed neutron at high energy, which enables breeding |

A pin cell is "fast" if the neutron spectrum has **no thermal peak**. We will *prove* that ours is fast with a flux-spectrum plot (Phase 1), compared against a water-cooled pin cell.

### 1.4 The one formula to keep in mind

With no leakage, $k_\infty$ is simply *neutrons produced* divided by *neutrons absorbed*, averaged over the neutron energy spectrum $\phi(E)$:

$$k_\infty = \frac{\displaystyle\int_V\!\int_0^\infty \nu\Sigma_f(\mathbf r,E)\,\phi(\mathbf r,E)\,\mathrm dE\,\mathrm dV}{\displaystyle\int_V\!\int_0^\infty \Sigma_a(\mathbf r,E)\,\phi(\mathbf r,E)\,\mathrm dE\,\mathrm dV},$$

where $\nu$ is the mean number of neutrons released per fission, $\Sigma_f$ is the macroscopic fission cross section, $\Sigma_a$ is the macroscopic absorption cross section, and $V$ is the cell volume.

**Every parameter we vary changes $k_\infty$ in only three ways:**

1. **More production (numerator):** more fissile material (higher Pu content), or more fast fission of ²³⁸U/²⁴⁰Pu.
2. **Less parasitic absorption (denominator):** absorption in cladding, coolant, oxygen, nitrogen or fission-product-like nuclides.
3. **A shifted spectrum $\phi(E)$:** a harder (higher-energy) spectrum raises $\eta = \nu\Sigma_f/\Sigma_a$ of ²³⁹Pu, while a softer spectrum lowers it in the fast range and increases ²³⁸U resonance capture.

The analysis (Phase 5) explains every result through these three mechanisms. That turns "here is a plot" into actual physics.

### 1.5 Our tool: Serpent, a Monte Carlo code

Serpent follows millions of individual neutrons through the geometry, sampling random collisions from nuclear-data cross sections. A **criticality calculation** runs in *cycles* (generations):

- **Inactive cycles** are discarded. They let the fission source settle into its true spatial shape (lecture 7, "Convergence of the fission source").
- **Active cycles** are averaged to give $k$ and its statistical standard deviation $\sigma_k$.
- The uncertainty shrinks only as $\sigma_k \propto 1/\sqrt{N}$, where $N$ is the number of active neutron histories. Halving the error costs **4×** the CPU time.

The cluster has **Serpent 1.1.18** (`sss`, single-core, about 2× faster per core) and **Serpent 2.1.31** (`sss2`, multi-threaded with OpenMP). Section 4.4 explains which one to use.

---

## 2. The cluster in one page

Source: `cluster.pdf` (MEGARAPTOR/MICRORAPTOR manual, ver. 2026-07-16).

```
 Your laptop ── ssh megaraptor ──►  MASTER NODE x25  ──── Slurm ────►  COMPUTE NODES
 (Claude Code,                       (login, editing,                    p2: n20 n21 n22   (oldest, slowest)
  scripts, git,                       short tests,                       p3: n30 n31 n32
  analysis)                           sbatch/squeue)                     p4: n40 n41 n42   (newest, fastest)
                                            │
                                   /users/<you>   ← your files; the ONLY place you may write
                                   /xsdata        ← nuclear data (read-only, NEVER copy)
                                   /share         ← software, examples.tar
```

**Key facts:**
- **Partitions on MEGARAPTOR are `p2`, `p3`, `p4`.** The example scripts in the manual use `p1`, which only exists on MICRORAPTOR. Copying them unchanged will fail.
- **Slurm account** is probably `mc` (course students). Phase 0 checks it with `sacctmgr show assoc user=$USER format=Account`.
- **Limits for `mc` on MEGARAPTOR:** 24 h per job, max 12 CPUs per job, max **30 CPUs** and 240 GB RAM across all running jobs *per user*. If our whole group shares one account, these limits are shared too.
- **Software:** `module load serpent/1.1.18` or `module load serpent/2.1.31`. Loading either also activates **serpentTools**, a Python parser for Serpent output. General Python lives in `source /share/conda/etc/profile.d/conda.sh && conda activate py314`.
- **Nuclear data:** ACE libraries for Serpent, for example `/xsdata/ACE/endfb7/endfb7.xsdata` (ENDF/B-VII.0) or `/xsdata/ACE/endfb7.1/endfb71.xsdata`. The manual recommends older libraries: faster and less RAM.
- **No backups.** Files are deleted when the account expires (`chage -l $USER`). Keep everything in git on our laptops too.

### 2.1 Rules we must never break (from manual section 7)

Breaking these gets jobs killed and partitions revoked. Repeated violations get the account deactivated.

1. **Master node:** at most **one** process (test run) per account at a time, at most **90 min**, and only if `free -g` shows **≥ 60 GB free**. Never run anything parallel there.
2. **Before submitting**, measure time and RAM with a short test on the master node (Section 2.2).
3. **RAM request ≤ 1.5 × measured.** Always pass `--mem`.
4. **`--cpus-per-task` must equal the threads actually used.** Serpent 1 always uses 1 CPU, so requesting more with Serpent 1 is an instant violation.
5. **Never copy `/xsdata`** to your home directory.
6. **No background servers or long-running processes** on the cluster (no VS Code server, Cursor server and so on). This is why Claude Code runs on the laptop (Section 3).
7. Only nuclear-engineering work.

### 2.2 Sizing a job (manual sections 6.2 and 7)

Run a short test on the master node with the **same neutron population** but **far fewer cycles**:

```bash
/usr/bin/time -v sss sfr_mox_pincell    # read "Maximum resident set size (kbytes)" and "Elapsed"
```

**RAM**, with $M_{\text{test}}$ the measured peak memory in GB and $N_{\text{cores}}$ the threads used:

$$M_{\text{req}} = 1.25 \times M_{\text{test}} \times \bigl(1 + 0.08\,(N_{\text{cores}} - 1)\bigr).$$

**Wall time.** The manual says to assume the slowest node under full load. The master x25 has single-thread S-score $2.18$, and the slowest p2 node under load has L-score $0.57$. So, with $t_{\text{test}}$ the test run time, $C$ the number of cycles and a 1.5× safety margin:

$$T_{\text{req}} \approx 1.5 \times t_{\text{test}} \times \frac{C_{\text{full}}}{C_{\text{test}}} \times \frac{2.18}{0.57}.$$

If we restrict jobs to `p3,p4` (L-scores about 1.0), replace $0.57$ with $0.97$.

---

## 3. How we work with Claude Code

**Claude Code runs on the laptop, inside this folder, and drives the cluster over SSH.** Do *not* install or run Claude Code on the cluster:
- rule 9 forbids long-lived background processes and personal servers there,
- rule 3 allows one master-node process per account, and a Claude session would occupy it,
- the laptop is where our git repository, Python analysis and report live anyway.

The loop looks like this:

```
 laptop: edit generator/scripts ──► rsync to cluster ──► ssh megaraptor 'sbatch …'
    ▲                                                          │
    └──── analyse/plot locally ◄── rsync results back ◄── squeue until done
```

- **Commands:** `ssh megaraptor '<command>'`. Every call is short and returns.
- **Upload:** `rsync -av --exclude-from=.rsyncignore ./serpent_project/ megaraptor:~/serpent_project/`
- **Download:** `rsync -av --include='*/' --include='*_res.m' --include='*_det*.m' --include='*.png' --include='*.out' --include='*.err' --exclude='*' megaraptor:~/serpent_project/runs/ ./serpent_project/runs/`
- **Waiting for jobs:** Claude Code may poll `squeue` *from the laptop* every few minutes. Never start a monitoring loop *on* the cluster.
- **Debug runs on the master node** must self-terminate: `timeout 120 sss <input>` instead of "start it, then press Ctrl+C" (Claude Code can't press Ctrl+C). `sss -plot <input>` only plots the geometry and exits [verify flag in manual].
- **Group members without Claude Code** can mount the cluster home on their own laptop with `sshfs` (manual section 4.4), or just pull the git repo.

**Sharing:** put `serpent_project/` in a private GitHub repo shared with the group. The cluster copy is just a mirror that receives files via rsync.

---

## 4. The baseline model

### 4.1 Physical choice: sodium-cooled fast reactor (SFR), MOX fuel, hexagonal pin cell

This is the most standard fast-reactor configuration and the one with the most published reference data. Reference design: the **OECD/NEA SFR benchmark "MOX-3600" core** (NEA/NSC/R(2015)9, *Benchmark for Neutronic Analysis of Sodium-cooled Fast Reactor Cores with Various Fuel Types and Core Sizes*). That report also defines oxide, carbide and metallic cores, which is useful for our fuel-type comparison. Claude Code could not fetch it automatically, so **download it manually** from the OECD-NEA website and put it in `docs/`.

### 4.2 Baseline parameters

| Quantity | Symbol | Baseline value | Source / note |
|---|---|---|---|
| Lattice | – | hexagonal cell, reflective boundary | Serpent surface `hexyc`, `set bc 2` |
| Fuel pellet outer radius | $r_{\text{f}}$ | 0.4715 cm | ESFR/MOX-3600 cold dimensions (9.43 mm OD) [verify] |
| Cladding inner radius | $r_{\text{ci}}$ | 0.4865 cm | 9.73 mm ID [verify] |
| Cladding outer radius | $r_{\text{co}}$ | 0.5365 cm | 10.73 mm OD [verify] |
| Pin pitch (flat-to-flat of hex cell) | $p$ | ≈ 1.19 cm, so $P/D \approx 1.11$ | **[verify] in NEA/NSC/R(2015)9**; wire wrap smeared into cladding or coolant |
| Gap | – | modelled as `void` | He density negligible; state the assumption |
| Fuel | – | (U,Pu)O₂, ≈ 95 % TD, ρ ≈ 10.4 g/cm³ | [verify] |
| Pu content | $e_{\text{Pu}}$ | ≈ 15–18 wt% of heavy metal | [verify] (benchmark inner core value) |
| Pu isotopic vector | – | Pu-238/239/240/241/242 ≈ 2.5/54/25/9/9.5 wt% (illustrative LWR-reprocessed) | **replace with the benchmark vector** [verify] |
| Uranium | – | depleted, 0.25 wt% ²³⁵U | typical |
| Cladding | – | steel; AISI 316 approx. Fe 65.5 / Cr 17 / Ni 12 / Mo 2.5 / Mn 2 / Si 1 wt%, ρ ≈ 7.9 g/cm³ | or the benchmark's ODS/15-15Ti [verify] |
| Coolant | – | sodium, ρ ≈ 0.85 g/cm³ at ≈ 700 K | [verify] |
| Temperatures | – | fuel ≈ 1200 K, cladding and coolant ≈ 600–900 K | choose the nearest library temperature (Section 4.3) |

Useful derived quantities, all printed by the generator script, where $A_{\text{cell}}$ is the hexagonal cell area:

$$A_{\text{cell}} = \frac{\sqrt3}{2}\,p^2,\qquad V_{\text{fuel}} = \frac{\pi r_{\text{f}}^2}{A_{\text{cell}}},\qquad V_{\text{cool}} = \frac{A_{\text{cell}} - \pi r_{\text{co}}^2}{A_{\text{cell}}},\qquad \frac{P}{D} = \frac{p}{2\,r_{\text{co}}}.$$

### 4.3 Nuclear data and temperatures

- Start with **ENDF/B-VII.0** (`/xsdata/ACE/endfb7/endfb7.xsdata`): fast and light on RAM, as the manual advises.
- In the usual Serpent ACE libraries, the suffix encodes temperature: `.03c` ≈ 300 K, `.06c` ≈ 600 K, `.09c` ≈ 900 K, `.12c` ≈ 1200 K, `.15c` ≈ 1500 K, `.18c` ≈ 1800 K. **Phase 0 checks which suffixes actually exist** for the nuclides we need.
- A fast spectrum needs **no thermal scattering libraries** (`therm` cards), since there is no hydrogen. **Exception:** the water-cooled *contrast* case must use one (light water S(α,β), for example `lwtr`/`lwj3`), otherwise its thermal spectrum is wrong. Phase 0 finds its name in the library file.

### 4.4 Serpent 1 or Serpent 2?

- **Default: Serpent 1 (`sss`)** for the parameter sweeps. Every case is a small independent pin cell, so we run many 1-CPU jobs in a **job array**, and Serpent 1 is about 2× faster per core.
- **Use Serpent 2 (`sss2`)** if we need a feature Serpent 1 lacks (for example on-the-fly Doppler broadening with the `tmp` entry [verify which version supports it], better output parameters, or OpenMP for one large case).
- **Quality check in Phase 1:** run the baseline in both codes. $k_\infty$ must agree within about $3\sigma$.

---

## 5. Research question and parameter study

### 5.1 Proposed research question (agree on it with the supervisor)

> **Which design and material parameters of a sodium-cooled fast-reactor fuel pin cell affect its infinite multiplication factor $k_\infty$ the most, and through which physical mechanism: neutron production, parasitic absorption or spectrum hardening and softening?**

Supporting questions:
1. How does $k_\infty$ depend on the **plutonium content** of the fuel?
2. How do the **fuel chemistry** (oxide, nitride, carbide, metal) and the **coolant** (Na, Pb, Pb–Bi, He) change $k_\infty$ and the spectrum?
3. How does the **lattice tightness** ($P/D$) affect $k_\infty$, and at what point does the cell stop being "fast"?
4. What is the **sodium void effect** (losing coolant), and why is it positive in an infinite lattice?
5. How large is the **Doppler effect** (fuel temperature) in a fast spectrum, compared with a thermal one?

Pick about 4–6 of the sweeps below. More is not better; a well-explained, statistically sound result per sweep is what earns the grade.

### 5.2 Sweep matrix (one parameter at a time, everything else at baseline)

| ID | Parameter | Values | Cases | What it teaches |
|---|---|---|---|---|
| **A** | Pu content $e_{\text{Pu}}$ (wt% of HM) | 10, 12, …, 30 | 11 | Production term; the critical Pu content for $k_\infty$ |
| **B** | Pitch → $P/D$ | 1.05, 1.10, 1.15, 1.2, 1.3, 1.4, 1.5, 1.6 | 8 | Fuel-to-coolant ratio; spectrum softening |
| **C** | Sodium density (% of nominal) | 100, 90, …, 0 | 11 | **Void reactivity**, a key SFR safety parameter |
| **D** | Coolant type | Na, Pb, Pb–Bi (LBE), He (≈ 7 MPa), void; **H₂O as thermal contrast** | 6 | Moderation versus absorption of coolants; shows clearly "fast versus slow" |
| **E** | Fuel chemistry (same $e_{\text{Pu}}$) | oxide, nitride (natural N), nitride (¹⁵N-enriched), carbide, U-Pu-10Zr metal | 5 | Light atoms soften the spectrum; ¹⁴N(n,p) is a parasitic absorber; heavy-metal density differs |
| **F** | Cladding material | SS316, HT9, ODS steel, SiC; Zircaloy-4 (thermal-reactor contrast) | 5 | Parasitic absorption in structural materials |
| **G** | Fuel temperature | 300, 600, 900, 1200, 1500, 1800 K (library suffixes) | 6 | Doppler effect, Doppler constant $K_D$ |
| **H** | Pu isotopic vector | e.g. "low Pu-240" versus "high Pu-240" (fresh versus multi-recycled Pu) | 2–3 | Fissile versus fertile Pu isotopes |
| **I** | Nuclear data library | ENDF/B-VII.0, VII.1, JEFF-3.1.1, JEFF-3.3 | 4 | Systematic (data) uncertainty, which belongs in the Discussion |
| **J** | Thermal reference | the same sweeps G and C for a PWR-like UO₂ pin cell | 6–11 | Contrast: Doppler and void behave very differently |

That gives about 60 cases for A–I. Each pin cell run takes minutes, so the whole study is a few hours of the 30-CPU allowance.

**Fuel-type caution (E):** changing chemistry changes *both* the light-atom content *and* the heavy-metal density (roughly oxide 11, carbide 13.6, nitride 14.3, metal 15.8 g/cm³ theoretical [verify]). To separate the two effects, run each fuel type twice: at its realistic density, and at the same heavy-metal atom density as the oxide.

### 5.3 How we turn $k$ values into results

**Reactivity** (in pcm, where 1 pcm = $10^{-5}$):

$$\rho = \frac{k - 1}{k},\qquad \Delta\rho_{1\to2} = \Bigl(\frac{1}{k_1} - \frac{1}{k_2}\Bigr)\times 10^5\ \text{pcm}.$$

**Its statistical uncertainty**, where $\sigma_1$ and $\sigma_2$ are the standard deviations of $k_1$ and $k_2$ (independent runs):

$$\sigma_{\Delta\rho} = 10^5\sqrt{\frac{\sigma_1^2}{k_1^4} + \frac{\sigma_2^2}{k_2^4}}\ \text{pcm}.$$

A difference counts as real only if $|\Delta\rho| > 3\,\sigma_{\Delta\rho}$.

**Sodium void worth:**

$$\Delta\rho_{\text{void}} = \rho(\text{voided}) - \rho(\text{nominal}).$$

**Doppler constant**, where $T_1$ and $T_2$ are two fuel temperatures:

$$K_D = \frac{\mathrm d\rho}{\mathrm d(\ln T)} \approx \frac{\rho(T_2) - \rho(T_1)}{\ln(T_2/T_1)}.$$

**Normalised sensitivity** (comparable across parameters $x$ with different units):

$$S_x = \frac{\Delta k / k}{\Delta x / x}.$$

A "tornado" bar chart of $S_x$ (or of $\Delta\rho$ for a ±10 % change in each parameter) answers the main research question in one figure.

### 5.4 Mechanism decomposition (the "why")

In each run, use Serpent **detectors** to tally, per material (fuel, cladding, coolant):
- absorption rate $A_m$,
- ν-fission rate $P_m$ (fuel only),
- flux spectrum $\phi(E)$ on an equal-lethargy grid (for example 300 bins from $10^{-11}$ to 20 MeV),
- fission rate versus energy, giving the fraction of fissions above 100 keV, below 0.625 eV, and in between.

Then:

$$k_\infty \approx \frac{\sum_m P_m}{\sum_m A_m}.$$

This must reproduce Serpent's own $k_\infty$ within statistics. It is an excellent **self-check** and the basis of every physical explanation ("replacing steel by SiC cut cladding absorption by X %, which raised $k_\infty$ by Y pcm").

**Why the sodium void is positive here:** removing sodium removes some (weak) moderation, so the spectrum hardens. That raises $\eta$ of ²³⁹Pu and the fast fission of ²³⁸U/²⁴⁰Pu, and it also removes Na capture. In a real finite core, voiding also *increases leakage*, which pushes $\rho$ down. Our infinite cell has no leakage, so it shows only the positive part. **Optional extension:** a finite 3D column (axial vacuum boundaries, same pin) shows how leakage changes the sign. Serpent supports this with the 2D hex boundary plus two `pz` planes (axial boundaries become vacuum).

---

## 6. Code structure

All file, function and variable names are **descriptive** (for example `pu_content_wt_percent`, not `x1`). Comments are minimal; the names and short docstrings carry the meaning.

```
Projektet/
├── PLAN.md                     ← this file
├── CLAUDE.md                   ← rules for Claude Code
├── docs/                       ← Serpent manual(s), NEA benchmark PDF, course PDFs
├── notes/
│   ├── cluster_inventory.md    ← Phase 0 output
│   └── decisions_log.md        ← every modelling choice + reason + source (feeds the Methods section)
└── serpent_project/            ← git repo; mirrored to megaraptor:~/serpent_project
    ├── model/
    │   ├── materials.py              ← compositions: mox_fuel(pu_content, pu_vector, density), sodium(density_fraction), ...
    │   ├── pin_cell_case.py          ← @dataclass PinCellCase with every parameter + baseline defaults
    │   └── serpent_input_writer.py   ← build_serpent_input(case) -> str   (the ONLY place Serpent syntax lives)
    ├── sweeps/
    │   └── sweep_definitions.py      ← sweep A–J as lists of PinCellCase
    ├── generate_cases.py             ← writes runs/<sweep>/<case_id>/<case_id> + runs/<sweep>/case_list.txt
    ├── slurm/
    │   └── run_serpent_array.sh      ← job-array template (Appendix B)
    ├── submit_sweep.sh               ← rsync up + sbatch for one sweep
    ├── collect_results.py            ← parse *_res.m and *_det*.m -> results/all_cases.csv
    ├── analysis/
    │   ├── plot_k_vs_parameter.py
    │   ├── plot_flux_spectra.py
    │   ├── plot_sensitivity_tornado.py
    │   └── reactivity_tools.py       ← rho, delta_rho_pcm, sigma_delta_rho_pcm, doppler_constant
    ├── inputs_handwritten/
    │   └── sfr_mox_pincell_baseline  ← Phase 1 hand-written input (the reference)
    ├── tests/
    │   └── test_generated_baseline_matches_handwritten.py
    ├── runs/                    ← generated inputs + Serpent outputs (git-ignored except case lists)
    └── results/                 ← all_cases.csv, figures (PDF, vector)
```

**Results table** (`results/all_cases.csv`), one row per run:
`case_id, sweep_id, varied_parameter, parameter_value, k_inf, k_inf_sigma, serpent_version, xs_library, neutrons_per_cycle, active_cycles, inactive_cycles, fast_fission_fraction, thermal_fission_fraction, mean_energy_of_fission_neutrons_MeV, absorption_fuel, absorption_clad, absorption_coolant, nu_fission_total, wall_time_s, slurm_job_id`

---

## 7. Phases

Suggested split for a group of four. Pair people so that at least two members understand each part (as in our project plan's strategy):

| Role | Owns |
|---|---|
| **Cluster and automation** (William + 1) | Phases 0, 2, 4; Claude Code driver |
| **Physics and data** | Section 4 numbers, material compositions, citations, `decisions_log.md` |
| **Statistics and analysis** | Phases 3, 5; plots |
| **Report** | Phase 6; theory sections from Section 1, writing-guide compliance |

### Phase 0: Reconnaissance and setup (½ day)

**Goal:** know exactly what the account, cluster and libraries offer before writing any physics.

Steps (Claude Code, **read-only** commands over SSH):
1. Account and limits: `hostname; whoami; chage -l $USER; sacctmgr show assoc user=$USER format=Account%20,Partition%20; sinfo; du -sh ~`.
2. What is already there: `ls -la ~` and `find ~ -maxdepth 3 -type f -not -path '*/.*' | head -200`. Look for the earlier `pincell_fresh`/`pincell_pilot` inputs (their geometry plots are in this folder) and the `bwr` example.
3. Software: `module avail 2>&1 | head -80`, then `module load serpent/1.1.18 && which sss sss2 && python -c "import serpentTools, sys; print(serpentTools.__version__, sys.version)"`.
4. Library temperatures and nuclides: grep the `.xsdata` file for `92235`, `92238`, `94238`–`94242`, `95241`, `8016`, `7014`, `7015`, `6000`/`6012`, `11023`, `82208`, `83209`, `2004`, `26056`, `24052`, `28058`, `42098`, `40090`, `14028`. List which suffixes (`.03c`…`.18c`) exist. Also find the light-water S(α,β) name (`grep -i -E 'lwtr|lwj|hh2o'`).
5. `tar -tf /share/examples.tar | head -50`, then copy them to `~/examples` as the manual says.
6. Master-node RAM check command: `free -g`.
7. Get the Serpent 1 manual (`https://serpent.vtt.fi/download/Serpent_manual.pdf`) and the NEA benchmark into `docs/`. Bookmark the Serpent 2 wiki (`https://serpent.vtt.fi/mediawiki/`).
8. Create `serpent_project/` with the structure in Section 6, `git init`, private GitHub repo, `.gitignore`, `.rsyncignore`.

**Done when:** `notes/cluster_inventory.md` lists the Slurm account, partitions, library paths with available temperature suffixes, the S(α,β) name, existing files, the account expiry date, and the serpentTools version.

> **Prompt:** *Read PLAN.md and CLAUDE.md. Do Phase 0 using only read-only commands over `ssh megaraptor`. Write the findings to notes/cluster_inventory.md, set up serpent_project/ as in Section 6, and tell me anything in the plan that the inventory contradicts. Do not submit jobs or run Serpent.*

### Phase 1: Hand-written baseline and "is it fast?" (1–2 days)

**Goal:** one correct, understood Serpent input for the baseline SFR pin cell, run once on a compute node. We write this one *by hand* (with Claude Code explaining each line) so the whole group understands the input format before we automate.

Steps:
1. Fill in the baseline numbers from the NEA benchmark (Physics role) and log each one with its source in `notes/decisions_log.md`.
2. Write `inputs_handwritten/sfr_mox_pincell_baseline` (skeleton in Appendix A).
3. Upload, then **plot the geometry first** (`sss -plot`). Check the plot visually: hexagon, four regions, correct radii. Compare with `pincell_fresh_geom1.png`.
4. Debug run on the master: `free -g` (≥ 60 GB free?), then `timeout 120 sss sfr_mox_pincell_baseline`.
5. Sizing test: the same population with about 10 % of the cycles, under `/usr/bin/time -v`. Compute `--mem` and `--time` with Section 2.2.
6. Submit one compute-node job (Appendix B with `--array=1-1`).
7. **Fast-spectrum proof:** run a water-cooled PWR-like pin cell (reuse the earlier `pincell_fresh` input if found) with the same flux-spectrum detector. Plot both spectra, $E\,\phi(E)$ versus $E$ on a log axis. The SFR must show no thermal peak, and most fissions must occur above about 100 keV.
8. Run the same baseline in Serpent 2 and check agreement within $3\sigma$.
9. Check the detector balance $\sum P / \sum A \approx k_\infty$.

**Done when:** we have $k_\infty \pm \sigma$ for the baseline, a geometry plot, a spectrum comparison plot (fast versus thermal), a measured RAM/time per $10^6$ histories, and a Serpent 1 versus 2 comparison. Sanity-check $k_\infty$ against the benchmark's own pin-cell results or the Serpent validation fast pin cell, if available.

> **Prompt:** *Do Phase 1 of PLAN.md. Explain every line of the Serpent input you write, as if to a group member who has never seen Serpent. Stop and show me the geometry plot before running anything else. Follow CLAUDE.md strictly for the master-node test and the sizing calculation.*

### Phase 2: Automation (1–2 days)

**Goal:** turn the hand-written input into a generator, so a sweep is one function call.

Steps:
1. `PinCellCase` dataclass with all parameters and baseline defaults. `case_id` is built from the varied parameter, e.g. `C_sodium_density_070pct`.
2. `materials.py`: compute mass fractions from $e_{\text{Pu}}$, the Pu vector, the U enrichment and the chemistry (O/N/C/Zr stoichiometry), using atomic masses. Unit-test that fractions sum to 1.
3. `serpent_input_writer.py`: the only file that knows Serpent syntax. It picks the library suffix from the temperature, and adds `therm` only for water.
4. **Test:** the generated baseline must equal the hand-written baseline (same materials and geometry), and must give the same $k_\infty$ within $3\sigma$.
5. `generate_cases.py`, `run_serpent_array.sh`, `submit_sweep.sh`, `collect_results.py`.
6. Dry-run sweep A with **tiny statistics** (e.g. 2000 neutrons × 20 cycles) to test the whole pipeline end to end.

**Done when:** `./submit_sweep.sh A` → jobs run → `python collect_results.py` → `results/all_cases.csv` has 11 rows with sensible numbers.

> **Prompt:** *Do Phase 2 of PLAN.md. Keep Serpent syntax confined to serpent_input_writer.py, use descriptive names and minimal comments, and finish with the low-statistics end-to-end dry run of sweep A.*

### Phase 3: Statistics and source convergence (½ day)

**Goal:** choose the neutron population and cycle counts so the uncertainties are small enough to resolve the effects we study.

1. Target $\sigma_k \lesssim 20$ pcm, so differences of about 100 pcm are significant. If a test with $N_{\text{test}}$ active histories gives $\sigma_{\text{test}}$, the required number of active histories is
   $$N_{\text{needed}} = N_{\text{test}}\left(\frac{\sigma_{\text{test}}}{\sigma_{\text{target}}}\right)^2.$$
2. Inactive cycles: plot $k$ per cycle (and Shannon entropy of the fission source if available [verify in Serpent 1/2]). A pin cell converges quickly; about 50 inactive cycles is likely plenty, but *show* it.
3. Fix one setting for all production runs, e.g. `set pop 20000 500 50` [to be decided from the test]. Log it.

**Done when:** a short note plus figure shows convergence, and the chosen settings give the target $\sigma$ at an acceptable run time.

### Phase 4: Production runs (1–3 days of wall time, mostly waiting)

1. Run the sweeps chosen with the supervisor. Throttle each array to the CPU limit (`--array=1-N%30`).
2. After each sweep: `collect_results.py`, quick plot, check all jobs exited cleanly (`grep -il error logs/*`).
3. Commit results to git. The cluster has no backups.

**Done when:** every chosen sweep has complete rows in `all_cases.csv` with $\sigma$ at target.

### Phase 5: Analysis (2–3 days)

Figures (all **vector PDF**, axis labels with units, consistent symbols):
1. $k_\infty$ versus each parameter, with $\pm 2\sigma$ error bars and $k_\infty = 1$ marked.
2. Spectra: SFR baseline versus voided versus Pb-cooled versus H₂O-cooled.
3. Absorption breakdown (fuel/cladding/coolant) per case, as stacked bars.
4. Void worth versus void fraction (sweep C); Doppler $\rho$ versus $\ln T$ with fitted $K_D$ (sweep G), for both SFR and PWR cells.
5. **Tornado chart** of sensitivities. This is the headline figure for the research question.

**Done when:** every figure has a one-paragraph physical explanation using the three mechanisms of Section 1.4.

### Phase 6: Report and presentation

Follow `Stylistic_Principles.pdf` (Section 9 below). Sections: Abstract, Introduction, Methods (model, Serpent, cluster, statistics), Results, Discussion and conclusions, References.

---

## 8. Quality checks (do not skip)

| Check | How | When |
|---|---|---|
| Geometry correct | `sss -plot`, then inspect the PNG | every new geometry type |
| Material compositions | fractions sum to 1; densities match the source; print atom densities and compare with the benchmark tables | Phase 1, 2 |
| Generator equals hand input | automated test | Phase 2 |
| Serpent 1 versus 2 | $k_\infty$ agree within $3\sigma$ | Phase 1 |
| Neutron balance | $\sum P/\sum A$ against $k_\infty$ | every run (in `collect_results.py`) |
| Fast spectrum | fast fission fraction > 0.9 [expectation; confirm] | every SFR case |
| Source converged | $k$ per cycle and entropy plot | Phase 3 |
| Significance | $\lvert\Delta\rho\rvert > 3\sigma_{\Delta\rho}$ before claiming an effect | analysis |
| Physical plausibility | $k_\infty$ rises with Pu content; voiding raises $k_\infty$ (infinite SFR cell); Doppler lowers $k_\infty$; water makes the spectrum thermal | analysis |
| Peer review | another member reviews every input change | always |

---

## 9. Report checklist (from `Stylistic_Principles.pdf`)

- $k_{\mathrm{eff}}$ / $k_\infty$: the "eff" is upright, the $k$ italic. Use **one** symbol consistently in text *and* plots.
- Numbers: `2.3×10⁻⁵`, never `2.3e-5`. Decimal point, not comma. Few decimals, matched to $\sigma$ (e.g. $k_\infty = 1.2345 \pm 0.0002$, not 1.234517).
- Isotopes as ²³⁹Pu (superscript mass number), never `Pu239`. Element *names* lowercase (plutonium, sodium).
- Figures: vector PDF, numbered, caption below ending with a full stop, referenced in the text, axis labels with units, font similar to body text.
- Tables: numbered title *above*, units in headers, referenced in the text.
- Equations are part of sentences; "where" below them in lowercase; define every symbol; use × not \*.
- Active voice ("we simulated…"). No contractions. Consistent "Serpent" spelling.
- Methods detailed enough to reproduce: code version, library, temperatures, population and cycles, geometry and compositions. This is exactly what `decisions_log.md` collects.

---

## 10. Pitfalls we already know about

1. **Partition `p1` is MICRORAPTOR only**; MEGARAPTOR uses `p2`–`p4`.
2. **Group account = shared limits.** Only one master-node test at a time for the whole group, and 30 CPUs total. Coordinate in the group chat before running anything on the master node.
3. **Serpent 1 with `--cpus-per-task` > 1** is a rule violation.
4. **Jobs must run from inside `~`**, otherwise Serpent can't write output.
5. **Windows line endings:** run `dos2unix` on files edited on Windows.
6. **Missing temperature suffix** in the library makes Serpent stop with an error. Check Phase 0's inventory before choosing temperatures.
7. **Water contrast case needs `therm`** (S(α,β)), otherwise the thermal spectrum is wrong.
8. **$k_\infty \neq k_{\mathrm{eff}}$:** don't present an infinite-cell void effect as "the reactor's" void coefficient without the leakage caveat.
9. **Confounded fuel-type comparison:** chemistry and density change together (Section 5.2).
10. **Overlapping error bars:** state $\sigma$ and test significance.
11. **Account expiry deletes everything:** `chage -l $USER`; git push results regularly.
12. **Plots exported as PNG:** the writing guide requires vector formats.

---

## 11. Glossary

| Term | Meaning |
|---|---|
| Pin cell | One fuel rod plus its share of coolant, repeated infinitely via reflective boundaries |
| $k_\infty$, $k_{\mathrm{eff}}$ | Multiplication factor without and with leakage |
| Reactivity $\rho$, pcm | $\rho = (k-1)/k$; 1 pcm = $10^{-5}$ |
| Spectrum | How neutron flux is distributed over energy; "hard" means fast neutrons dominate |
| MOX | Mixed oxide fuel, (U,Pu)O₂ |
| $P/D$ | Pitch-to-diameter ratio; small means tight lattice and little coolant |
| Void effect | Change in reactivity when coolant is lost |
| Doppler effect | Hotter fuel broadens ²³⁸U resonances, giving more capture and lower $k$ |
| Active/inactive cycles | Monte Carlo generations that are averaged / discarded |
| Detector | Serpent's name for a user-defined tally (a reaction rate or flux in a region and energy range) |
| Slurm, sbatch, job array | The cluster's job scheduler; the submit command; many similar jobs from one script |
| Master/compute node | Login machine (no heavy work) / machines that run the jobs |

---

## Appendix A: Serpent input skeleton (baseline, to be completed in Phase 1)

All syntax here is **[verify] against the Serpent manual**. Serpent 1 and 2 share most of it. Fuel fractions below correspond to 18 wt% Pu in heavy metal with the illustrative Pu vector; `materials.py` computes the real ones.

```
set title "SFR MOX hexagonal pin cell - baseline"

% ---------- geometry (cm) ----------
pin fuel_pin
fuel_mox     0.4715
void         0.4865
clad_ss316   0.5365
sodium

surf hex_cell_boundary hexyc 0.0 0.0 0.595
% hexyc parameter = half of the pin pitch (flat-to-flat / 2), here p = 1.19 cm [verify pitch]

cell pin_cell_interior 0 fill fuel_pin -hex_cell_boundary
cell outside_world     0 outside       hex_cell_boundary

% ---------- materials (negative = mass fractions / g/cm3) ----------
mat fuel_mox  -10.4
92235.12c  -0.0018
92238.12c  -0.7210
94238.12c  -0.0040
94239.12c  -0.0857
94240.12c  -0.0397
94241.12c  -0.0143
94242.12c  -0.0151
8016.12c   -0.1185

mat clad_ss316 -7.9
26056.09c  -0.655
24052.09c  -0.170
28058.09c  -0.120
42098.09c  -0.025
25055.09c  -0.020
14028.09c  -0.010

mat sodium -0.85
11023.09c  1.0

% ---------- physics and run settings ----------
set acelib "/xsdata/ACE/endfb7/endfb7.xsdata"
set bc 2
set pop 20000 300 50
plot 3 800 800

% ---------- detectors (tallies) ----------
ene equal_lethargy_grid 3 300 1.0E-11 20.0
det flux_spectrum      de equal_lethargy_grid
det fission_spectrum   de equal_lethargy_grid dr -6 fuel_mox
det absorption_fuel    dr -2 fuel_mox
det absorption_clad    dr -2 clad_ss316
det absorption_coolant dr -2 sodium
det nu_fission_fuel    dr -7 fuel_mox
% response numbers (-2 capture/absorption, -6 fission, -7 nu-fission) and ene type 3 [verify]
```

## Appendix B: Slurm job-array script (MEGARAPTOR)

```bash
#!/bin/bash
#SBATCH --account=mc                  # confirm in Phase 0
#SBATCH --job-name=fastpin_SWEEPID
#SBATCH --output=logs/%x_%A_%a.out
#SBATCH --error=logs/%x_%A_%a.err
#SBATCH --partition=p2,p3,p4          # NOT p1 (that is MICRORAPTOR)
#SBATCH --array=1-NCASES%30           # %30 = never more than 30 running at once (per-user CPU cap)
#SBATCH --cpus-per-task=1             # Serpent 1 is single-core: must be 1
#SBATCH --mem=MEM_FROM_SIZING_TEST    # Section 2.2, <= 1.5x measured
#SBATCH --time=TIME_FROM_SIZING_TEST  # Section 2.2

module purge
module load serpent/1.1.18
ulimit -s unlimited

case_id=$(sed -n "${SLURM_ARRAY_TASK_ID}p" case_list.txt)
cd "$case_id"
srun sss "$case_id"
```

Submit from `~/serpent_project/runs/<sweep>/`, after `mkdir -p logs`.

For a Serpent 2 OpenMP run: `module load serpent/2.1.31`, `--cpus-per-task=N` (N ≤ 12, about 6–10 is useful), `export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK`, `srun sss2 -omp $SLURM_CPUS_PER_TASK <input>` [verify the `-omp` flag; the manual example relies on `OMP_NUM_THREADS` only].

## Appendix C: Sources to cite

- J. Dufek, *MEGARAPTOR and MICRORAPTOR NSE Division HPC Clusters, User's Manual*, KTH, ver. 2026-07-16.
- J. Leppänen et al., *The Serpent Monte Carlo code: Status, development and applications in 2013*, Ann. Nucl. Energy 82 (2015) 142–150. Also cite the Serpent 1 user manual and the Serpent wiki.
- OECD/NEA, *Benchmark for Neutronic Analysis of Sodium-cooled Fast Reactor Cores with Various Fuel Types and Core Sizes*, NEA/NSC/R(2015)9, 2016.
- The nuclear data library used (e.g. ENDF/B-VII.0: M. B. Chadwick et al., Nucl. Data Sheets 107 (2006) 2931).
- A reactor physics textbook for theory (e.g. Lamarsh & Baratta, *Introduction to Nuclear Engineering*, as recommended in the writing guide).
