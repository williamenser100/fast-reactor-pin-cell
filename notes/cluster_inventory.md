# Cluster inventory (Phase 0)

Collected on 2026-11-11 from the laptop with read-only commands over `ssh megaraptor`.
No Serpent runs, no job submissions, no file changes on the cluster.
The cluster manual referred to below is `cluster.pdf` (ver. 2026-07-16), and the Serpent manual is `docs/Serpent_manual.pdf` (Serpent 1 User's Manual, 18 June 2015).

## 1. Account

| Item | Value | Command |
|---|---|---|
| Master node | `x25` | `hostname` |
| Linux user | `mp10` (uid 1006, group `mp10` only) | `whoami; id` |
| Home directory | `/users/msc/mp10` | `echo ~` |
| SSH alias on laptop | `megaraptor` (user `mp10`; host, port and key are in each member's own `~/.ssh/config`, see the cluster manual) | `~/.ssh/config` |
| **Account expiry** | **2026-12-15** (password never expires) | `chage -l $USER` |
| Disk used | 2.9 MB | `du -sh ~` |
| Disk quota | 100 GB soft and hard, no file-count limit | `quota -s` |
| Filesystem | `/users/msc` on 1.9 TB NVMe, 6 % used | `df -h ~` |
| Warning files from the manager | none found in `~` | `ls ~` |

The whole group logs in as the single Linux user `mp10`.
Every "per user" and "per account" limit below is therefore shared by the whole group, including the rule of one master-node process at a time.

All files on the cluster are deleted when the account expires on 2026-12-15, 34 days from today.
Everything we want to keep must be pulled to the laptop and pushed to git before then.

## 2. Slurm accounts and limits

`sacctmgr show assoc user=$USER` gives two associations on cluster `megaraptor`:

| Slurm account | QOS | Time per job | Running jobs per user | Nodes per job | CPUs per job | RAM per job | CPUs per user | RAM per user | Partitions allowed |
|---|---|---|---|---|---|---|---|---|---|
| `mc` | `qosmc` | 24 h | no limit | 1 | 12 | 30 GB | **160** | 240 GB | p2, p3, p4 |
| `xmc` | `qosxmc` | 7 days | **1** | 1 | 16 | 50 GB | no limit | no limit | p3, p4 (not p2) |

Sources: `sacctmgr show qos qosmc,qosxmc` (both QOS have the `DenyOnLimit` flag, so a job over the limit is rejected at submission), `scontrol show partition` (AllowAccounts), and manual Table 2.3, which agrees exactly.

`mc` is the account to use for all our sweeps.
`xmc` only makes sense for a single job that needs more than 24 h.

## 3. Partitions and nodes (MEGARAPTOR)

| Partition | Node | CPUs | RAM in Slurm (MB) | L / S score (manual) | State on 2026-11-11 |
|---|---|---|---|---|---|
| p4 (default) | n40 | 32 | 61 000 | 1.12 / 2.18 | allocated |
| p4 | n41 | 32 | 91 000 | 1.12 / 2.18 | allocated |
| p4 | n42 | 32 | 91 000 | 1.12 / 2.18 | idle |
| p3 | n30 | 32 | 61 000 | 1.00 / 2.16 | idle |
| p3 | n31 | 32 | 61 000 | 0.97 / 2.11 | idle |
| p3 | n32 | 32 | 61 000 | 0.97 / 2.11 | idle |
| p2 | n20 | 28 | 122 000 | 0.57 / 1.18 | idle |
| p2 | n21 | 32 | 122 000 | 0.57 / 0.89 | **down** |
| p2 | n22 | 24 | 30 100 | 0.95 / 1.83 | idle |
| gpu | n01 | 24 | 91 000 | 0.79 / 1.56 | idle (not open to `mc`/`xmc`) |

Master node x25: 32 CPUs, 123 GB RAM, S-score 2.18.
Partition `p1` does not exist on MEGARAPTOR, as expected.
None of the partitions has its own time limit; the 24 h limit comes from the `mc` QOS.

Master-node memory at the time of the check (`free -g`): total 123 GB, used 3 GB, **free 9 GB**, buff/cache 110 GB, **available 119 GB**.
See open question Q1 at the end about which of these numbers the "≥ 60 GB free" rule refers to.

No jobs of ours were queued or running (`squeue -u $USER` was empty).

## 4. Software

| Item | Value |
|---|---|
| Module system | Lmod. `module` only exists in a **login shell**: plain `ssh megaraptor 'module ...'` fails with "command not found". Use `ssh megaraptor 'bash -lc "module ..."'`. |
| Modules | `serpent/1.1.18`, `serpent/2.1.31` (default), `abacas/0.0.5`, `abacas/0.1.0`, `abacas/0.1.1` (default) |
| `serpent/1.1.18` | prepends `/share/apps/serpent/1.1.18/bin` (contains `sss`) and the serpentTools venv |
| `serpent/2.1.31` | prepends `/share/apps/serpent/2.1.31/bin` (contains `sss2`, plus `p4-sss2` and `old-sss2`) and the serpentTools venv |
| `sss` without modules | `/usr/local/bin/sss`, byte-identical (md5) to the module's Serpent 1.1.18 |
| `sss2` without modules | `/usr/local/bin/sss2`, byte-identical (md5) to the module's Serpent 2.1.31 `sss2` |
| Version strings | `sss` reports `Serpent 1.1.18`; `sss2` contains `2.1.31` |
| **serpentTools** | **0.11.0**, on Python 3.12.3, in venv `/share/apps/python_envs/serpent_tools_env` (activated by either Serpent module) |
| Python outside modules | only `python3` (system); `python` is not on the PATH |
| Conda environments | `py312`, `py312-gpu`, `py314` under `/share/conda` |

Command-line options confirmed in the binaries:
- `sss`: `-plot` (stop after geometry plot), `-checkvolumes`, `-testgeom`, `-replay`, `-disperse`, `-version`, `-mpi`.
- `sss2`: `-plot`, `-omp`, `-mpi`, `-checkvolumes`, `-norun`, `-replay`, `-elem`, `-disperse`, `-version`.

## 5. Nuclear data

### 5.1 ACE libraries present

| Library | Directory listing file | Decay / fission-yield files | Temperature suffixes |
|---|---|---|---|
| ENDF/B-VII.0 | `/xsdata/ACE/endfb7/endfb7.xsdata` | `sss_endfb7.dec`, `sss_endfb7.nfy` | `.03c`-`.18c` = 300-1800 K |
| ENDF/B-VII.1 | `/xsdata/ACE/endfb7.1/endfb71.xsdata` | `endfb71.dec`, `endfb71.nfy` | `.03c`-`.18c` = 300-1800 K |
| JEFF-3.1.1 | `/xsdata/ACE/jeff311/jeff311.xsdata` | `sss_jeff311.dec`, `sss_jeff311.nfy` | `.03c`-`.18c` = 300-1800 K |
| JEFF-3.3 | `/xsdata/ACE/jeff33/jeff33.xsdata` | `jeff33.dec`, `sss_jeff33.dec`, `sss_jeff33.nfy` | `.03c` = **294 K**, `.06c`-`.18c` = 600-1800 K |
| ENDF/B-VIII.1 | `/xsdata/ACE/endfb8.1/endfb81.xsdata` | none | different scheme: `.10c`=294, `.11c`=600, `.12c`=900, `.13c`=1200, `.14c`=2500, `.15c`=0, `.16c`=233, `.17c`=273 K |
| JEFF-4.0 | `/xsdata/ACE/jeff4/jeff4.xsdata` | none | different scheme: `.40c`=294, `.41c`=300, `.42c`=600, `.43c`=900, `.44c`=1200, `.45c`=1500, `.46c`=1800 K |

The manual (Table 2.1) lists ENDF/B-VIII.1 and JEFF-4.0 ACE as "not present", but their `.xsdata` files now exist.
They have no decay or yield files and have not been tested with our Serpent versions, so we leave them out.

### 5.2 Temperature suffixes in ENDF/B-VII.0 for the nuclides we may need

Source: `awk` over `/xsdata/ACE/endfb7/endfb7.xsdata`, counting transport entries (column 3 = 1) whose alias equals the ZAID.
The file has 5182 transport entries and 108 S(α,β) entries.
Every ZAID has exactly the six suffixes `.03c` (300 K), `.06c` (600 K), `.09c` (900 K), `.12c` (1200 K), `.15c` (1500 K) and `.18c` (1800 K).
The only incomplete ZAID in the whole file is `2000` (natural helium, 5 of 6), which we do not need.

| Nuclide (ZAID) | .03c | .06c | .09c | .12c | .15c | .18c |
|---|---|---|---|---|---|---|
| ²³⁵U (92235), ²³⁸U (92238) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ²³⁸Pu-²⁴²Pu (94238-94242) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ²⁴¹Am (95241) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ¹⁶O (8016), ¹⁷O (8017) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ¹⁴N (7014), ¹⁵N (7015) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| natural C (6000), ¹²C (6012) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ²³Na (11023) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ²⁰⁴Pb, ²⁰⁶Pb, ²⁰⁷Pb, ²⁰⁸Pb (82204-82208), natural Pb (82000) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ²⁰⁹Bi (83209) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ³He (2003), ⁴He (2004) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Fe: 26054, 26056, 26057, 26058, natural 26000 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Cr: 24050, 24052, 24053, 24054, natural 24000 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Ni: 28058, 28060, 28061, 28062, 28064, natural 28000 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Mo: 42092-42100 (all seven), natural 42000 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Zr: 40090, 40091, 40092, 40094, 40096, natural 40000 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Si: 14028, 14029, 14030, natural 14000 | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| ⁵⁵Mn (25055) | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| Also present: ¹H, ²H, ¹⁰B, ¹¹B, ²⁷Al, ³¹P, ³²S, Ti (22000, 22048), ⁵¹V, ⁶³Cu, ⁹³Nb, ¹⁸¹Ta, ¹⁸⁴W | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |

### 5.3 The same nuclides in the other libraries (for sweep I)

All isotopes listed above exist at all six temperatures in ENDF/B-VII.1, JEFF-3.1.1 and JEFF-3.3.
The **natural-element** entries do not:

| ZAID | ENDF/B-VII.0 | ENDF/B-VII.1 | JEFF-3.1.1 | JEFF-3.3 |
|---|---|---|---|---|
| 6000 (C) | ✓ | ✓ | ✓ | ✓ |
| 6012 (¹²C) | ✓ | **missing** | ✓ | ✓ |
| 14000, 22000, 24000, 26000, 28000, 40000, 42000, 82000 | ✓ | **missing** | ✓ | **missing** |

To run the same materials in every library, write steels, zirconium and lead as isotopes, and carbon as `6000`.

### 5.4 Light-water S(α,β) libraries

| Library file | Water S(α,β) names | Temperatures |
|---|---|---|
| ENDF/B-VII.0 (`endfb7.xsdata`) | **`lwe7.00t`-`lwe7.18t`** (ENDF/B-VII.0 evaluation) | 294, 350, 400, 450, 500, 550, 600, 650, 800 K |
| ENDF/B-VII.0 (`endfb7.xsdata`) | also `lwe6.xxt` (ENDF/B-VI.8), `lwj2.xxt` (JEFF-2.2), `lwj3.xxt` (JEFF-3.1) | 294-1000 K |
| ENDF/B-VII.1 (`endfb71.xsdata`) | `lwtr.30t`-`lwtr.80t` | 294-800 K |
| JEFF-3.1.1 (`jeff311.xsdata`) | same set as ENDF/B-VII.0 (`lwe6`, `lwe7`, `lwj2`, `lwj3`) | 294-1000 K |
| JEFF-3.3 (`jeff33.xsdata`) | `hh2o.00t`-`hh2o.10t` | 294-1000 K |

Exact suffix-to-temperature map for `lwe7`: `.00t`=294 K, `.02t`=350 K, `.04t`=400 K, `.06t`=450 K, `.08t`=500 K, `.10t`=550 K, `.12t`=600 K, `.14t`=650 K, `.18t`=800 K.
All S(α,β) data for ENDF/B-VII.0 sit in one file, `/xsdata/ACE/endfb7/data/sssth1`.
The ENDF/B-VII.0 files also contain heavy water (`hwe6`, `hwe7`, `hwj2`, `hwj3`) and graphite (`gre6`, `gre7`, `grj2`, `grj3`).
For the water contrast case with ENDF/B-VII.0, use `therm lwtr lwe7.12t` (600 K) or the `lwe7` entry closest to the chosen water temperature.
`lwtr` in that card is our own label, not a library name.

## 6. `/share/examples.tar`

`tar -tvf /share/examples.tar` (40 KB, dated 2026-06-18):

| Path in the tar | What it is |
|---|---|
| `examples/Serpent1_single/bwr` | BWR assembly input (below) with `set pop 10000 100 10` |
| `examples/Serpent1_single/single.sh` | sbatch: account `mc`, partition `p4`, 1 CPU, 2 GB, 1 h, `module load serpent/1.1.18`, `srun sss bwr` |
| `examples/Serpent1_array/bwr_1`, `bwr_2`, `bwr_3` | three copies of the same BWR input |
| `examples/Serpent1_array/array.sh` | sbatch job array `--array=1-3`, `mc`, `p4`, 1 CPU, 2 GB, 1 h, `srun sss bwr_${SLURM_ARRAY_TASK_ID}` |
| `examples/Serpent2_parallel/bwr` | BWR input with `set pop 50000 20 5` |
| `examples/Serpent2_parallel/parallel.sh` | sbatch: `mc`, `p4`, 12 CPUs, 4 GB, 2 h, `module load serpent/2.1.31`, `export OMP_NUM_THREADS=$SLURM_CPUS_PER_TASK`, `srun sss2 bwr` (no `-omp` flag) |

The archive has already been copied to `~/examples.tar` and unpacked into `~/examples/` (done earlier by a group member), so nothing needed to be written.

## 7. Existing files in the cluster home directory

| Path | Date | What it is |
|---|---|---|
| `simple-model/pincell` | 2026-10-27 | UO₂ PWR pin cell **with depletion** (see 7.1) |
| `simple-model/pincell_fresh` | 2026-10-27 | byte-identical to `pincell`; produced `pincell_fresh_geom1.png` (same image as on the laptop); no transport output kept |
| `simple-model/pincell_pilot` | 2026-10-27 | same input with `set pop 5000 10 5` instead of `5000 200 50`; **ran to completion** (`_res.m`, `_dep.m`, `.out`, geometry plot) |
| `simones-mapp/pincell.inp` | 2026-10-27 | separate UO₂ pin cell, JEFF-3.1.1, no depletion (see 7.2); never run (no output files) |
| `anton_test/bwr` | 2026-11-06 | byte-identical copy of `examples/Serpent1_single/bwr` |
| `anton_test/single.sh` | 2026-11-06 | copy of the example sbatch script, but runs `sss anton_in`, a file that does not exist; error log name `errors:%j.txt` contains a colon; never run |
| `examples/` | 2026-06/10 | unpacked `examples.tar`, with outputs from earlier runs of all three Serpent 1 examples (see 7.3) |
| `examples.tar` | 2026-11-06 | copy of `/share/examples.tar` |
| `input-test.inp` | 2026-10-24 | empty file (0 bytes), plus a `.seed` from someone running `sss` on it |

### 7.1 `simple-model/pincell`, `pincell_fresh` and `pincell_pilot`

A 2D **square** UO₂ pin cell for a water-cooled reactor:
- Geometry: fuel radius 0.412 cm, `void` gap to 0.420 cm, cladding to 0.475 cm, water outside; `sqc` half-width 0.630 cm (pitch 1.26 cm); `set bc 2`.
- Fuel: UO₂ at 10.4 g/cm³, 4 at% ²³⁵U, nuclides at `.09c` (900 K), `burn 1`.
- Cladding: pure natural zirconium `40000.06c`, 6.55 g/cm³.
- Water: 0.70 g/cm³, `.06c`, `moder lwtr 1001` with `therm lwtr lwe7.12t` (600 K).
- Library: ENDF/B-VII.0, with decay and fission-yield files.
- Depletion: `set bumode 2`, `set pcc 1`, `set powdens 0.040` (40 kW/kgU), 14 steps to 900 days.
- `plot 3 800 800`.

Result of `pincell_pilot` (Serpent 1.1.18, 5000 neutrons, 10 active and 5 inactive cycles, so low statistics):
fresh fuel `IMP_KEFF` = 1.3596 ± 0.0021 (relative), `ANA_KEFF` = 1.3585, `SIX_FF_KINF` = 1.3588; 15 burnup points in total; Serpent's own `MEMSIZE` = 2787 MB, dominated by the depletion data.

This is the natural starting point for the thermal contrast case in Phase 1 step 7, after removing everything related to depletion (`burn 1`, `dep`, `bumode`, `pcc`, `powdens`, `declib`, `nfylib`).
Writing the cladding as Zr isotopes instead of `40000` would make it work in all libraries (Section 5.3).

### 7.2 `simones-mapp/pincell.inp`

Another square UO₂ pin cell (pitch 1.26 cm): fuel radius 0.4096 cm, helium gap (`2004.06c`, 1.8×10⁻⁴ g/cm³) to 0.4180 cm, Zr cladding written as isotopes to 0.4750 cm, water 0.72 g/cm³ with `therm lwtr lwj3.11t` (574 K), 4 % enrichment, JEFF-3.1.1, `set pop 5000 100 20`, no depletion, no plot card.

### 7.3 BWR example (`examples/*/bwr`)

The manual's example: an asymmetric 12 × 12 BWR assembly (pin pitch 1.295 cm, assembly pitch 15.375 cm) with seven UO₂ fuel types, one containing Gd, a central water channel, 40 % void in the coolant, JEFF-3.1.1 with `lwj3.11t`, group-constant generation (`set gcu 0`, `set nfg 4 ...`), a mesh plot and four fast-flux detectors.

Results from earlier runs in `~/examples`:

| Run | Population | `IMP_KEFF` | CPU time | Notes |
|---|---|---|---|---|
| `Serpent1_single/bwr` | 10000 × 100 active, 10 inactive | 1.06899 ± 0.00078 (relative) | 0.78 min | |
| `Serpent1_single/bwr_test` | 10000 × 20 active, 10 inactive | 1.06934 ± 0.00153 | 0.14 min | sizing test under `/usr/bin/time -v` |
| `Serpent1_array/bwr_2`, `bwr_3` | 10000 × 100, 10 | 1.06977, 1.06962 | about 0.7 min each | array job |

The sizing test log (`bwr_test.log`) shows a wall time of 8.5 s and a **maximum resident set size of 832 MB**, while Serpent's own `MEMSIZE` for the same run is only 215 MB.
So `MEMSIZE` underestimates the real memory use by about a factor of 4, and `--mem` must be based on `/usr/bin/time -v`, as the manual says.
Serpent 1 reports `TOT_CPU_TIME` and `RUNNING_TIME` in minutes.

## 8. Serpent features confirmed in the manual and binaries

| Feature | Serpent 1.1.18 | Evidence |
|---|---|---|
| Geometry-only run | `sss -plot <input>` | manual, list of command-line options; binary usage string |
| Doppler broadening | `mat <name> <dens> tmp <T>`; broadens only **upward** from the library temperature | manual §4.3; `DopplerBroad` in binary |
| Shannon entropy of the fission source | computed by default; mesh set with `set entr`; outputs `ENTROPY_X/Y/Z/TOT` | manual §5.13 |
| Equal-lethargy energy grid | `ene <name> 3 <N> <Emin> <Emax>` | manual §7.1.2 |
| Detector responses | `-1` total, **`-2` total capture**, `-3` elastic, `-5` (n,2n), `-6` fission, `-7` fission neutron production (ν-fission), `-8` fission energy | manual Table 7.2 |
| Detector domain | `dr <mt> <mat>` sets the **cross section**; the scoring region is set separately with `dm`, `dc` or `du` | manual Table 7.1 and §7.1.1 |
| Hexagonal boundary | `hexyc x0 y0 r`; manual recommends `set bc 3` (periodic); reflective (`bc 2`) works only with 30° symmetry and was enabled in 1.1.18 | manual §3.7 |

## 9. Commands used

```bash
ssh megaraptor 'hostname; whoami; id; chage -l $USER; sacctmgr -n -P show assoc user=$USER format=Cluster,Account,User,Partition,QOS; sinfo; du -sh ~; df -h ~; quota -s; free -g; squeue -u $USER'
ssh megaraptor 'sacctmgr -n -P show qos qosmc,qosxmc format=Name,MaxWall,MaxTRESPerJob,MaxTRESPerUser,MaxJobsPerUser,Flags; scontrol show partition; sinfo -N -o "%N %P %c %m %T"'
ssh megaraptor 'bash -lc "module avail; module load serpent/1.1.18; which sss sss2 python; python -c \"import serpentTools, sys; print(serpentTools.__version__, sys.version)\""'
ssh megaraptor 'awk '\''$1==$2 && $3==1 {split($2,a,"."); s[a[1]]=s[a[1]]" "a[2]"("$7")"} END{for(z in s) print z":"s[z]}'\'' /xsdata/ACE/endfb7/endfb7.xsdata | sort -n'
ssh megaraptor 'awk '\''$3==3 {print $2, $7}'\'' /xsdata/ACE/endfb7/endfb7.xsdata | sort -u'
ssh megaraptor 'tar -tvf /share/examples.tar; ls -la ~; find ~ -maxdepth 4 -not -path "*/.*"'
```

## 10. Open questions

- **Q1. "≥ 60 GB free" on the master node.** Right now `free -g` shows 9 GB in the `free` column but 119 GB `available` (most RAM is page cache that the kernel releases on demand). The manual says to check "free RAM" with `top`/`htop`. Read literally, no master-node test would be allowed today. We need the group, or the cluster manager, to decide which number counts before the first sizing test in Phase 1.
- **Q2. Shared login.** Everyone uses `mp10`, so agree in the group chat who runs master-node tests and when.
