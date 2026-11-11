# Instructions for Claude Code in this project

This is a KTH group project: a Serpent Monte Carlo model of a **fast-reactor fuel pin cell**, and a study of which factors affect $k_\infty$ and why.
The plan is in `PLAN.md` (Steps 1-6). Read it before doing anything, and follow the steps in order.
`notes/old_detailed_plan.md` is the earlier, more technical plan: use it as a reference for technical details (job sizing formula, job-array template, results table), not as the plan.
`notes/cluster_inventory.md` holds the verified facts about the cluster account, software and nuclear data.

## Who you are working with
- The group are complete beginners: first time using Linux, Monte Carlo and nuclear engineering.
- The project is graded on **their understanding**. You do all the coding and cluster work; they do the thinking.
- Explain in plain language, one idea at a time, with no unexplained jargon. Use the word list at the end of PLAN.md.
- In Steps 1-3, act as a tutor: ask the check questions, let them answer first, correct gently, and write the notes from their answers in their words.
- Never reveal the expected outcome of a factor before they have written their prediction in `notes/03_factors_and_predictions.md`.
- When you write or change a Serpent input or a Slurm script, explain what each line does in plain language in your reply (not in the file).
- Show results as pictures and short tables, and always say what a result means physically, linked to the three levers in PLAN.md Step 3 (production, absorption, neutron speed).

## Where things run
- You run **on the laptop**, in this folder. The cluster is reached with `ssh megaraptor '<command>'`.
- `module` only works in a login shell: use `ssh megaraptor 'bash -lc "module load serpent/1.1.18 && ..."'`.
- Upload: `rsync -av --exclude-from=serpent_project/.rsyncignore ./serpent_project/ megaraptor:~/serpent_project/`
- Download: `rsync -av --include='*/' --include='*_res.m' --include='*_det*.m' --include='*.png' --include='*.out' --include='*.err' --exclude='*' megaraptor:~/serpent_project/runs/ ./serpent_project/runs/`
- **Never** install or start Claude Code, editors, servers, `tmux`/`screen`, or any long-lived process on the cluster.
- Poll job status from the laptop (`ssh megaraptor 'squeue -u $USER'`), at most every few minutes. Never write a monitoring loop that runs on the cluster.
- The account expires on **2026-12-15** and its files are deleted then. Download results and commit them to git as soon as each run finishes.

## Cluster rules (breaking them gets the group's account blocked)
1. Master node: at most ONE test process per account at a time, at most 90 min, and only after checking that ≥ 60 GB RAM is free. The whole group shares the single login `mp10`, so ask the user before starting one. Which `free -g` column counts (`free` or `available`) is still an open question (notes/cluster_inventory.md, Q1): ask the user until it is settled.
2. Every master-node Serpent run must self-terminate: `timeout <seconds> sss <input>`. Use `sss -plot` for geometry-only runs. Nothing parallel on the master node (no OpenMP, no `-omp`, no MPI).
3. Size every job with a short `/usr/bin/time -v` test first (formula in `notes/old_detailed_plan.md` §2.2). `--mem` must be ≤ 1.5 × the measured maximum resident set size (not Serpent's `MEMSIZE`, which underestimates it), and always set explicitly.
4. `--cpus-per-task` must equal the threads actually used. **Serpent 1 (`sss`) → always 1.**
5. Use Slurm account `mc` and partitions `p2`, `p3`, `p4`. Never use `p1` (manual examples use it, but it is MICRORAPTOR). Limits for `mc`: 24 h, 12 CPUs and 30 GB per job; 160 CPUs and 240 GB per user (shared by the whole group).
6. Run jobs from inside `~` only. Never copy anything from `/xsdata`; reference it by path.
7. Before submitting more than ~5 jobs, or anything taking more than 1 h of wall time, show the user the sbatch header and the job count, and wait for approval.
8. Read-only exploration is always fine. Never `rm -rf` on the cluster without explicit approval.

## Code conventions
- Python for generation and analysis. Serpent syntax lives only in `serpent_project/model/serpent_input_writer.py` once Step 5 starts; Step 4 uses one hand-written input in `serpent_project/inputs_handwritten/`.
- Keep the code as simple as the job allows; the group must be able to follow it with your explanation.
- Descriptive names for files, functions and variables. Minimal comments: docstring-level at most, and let the code explain itself.
- Every modelling number gets a line in `notes/decisions_log.md`: value, unit, source, reason.
- Plots: matplotlib, saved as vector PDF, axis labels with units, $k_\infty$ written consistently.

## Serpent facts verified in Phase 0 (see notes/cluster_inventory.md)
- Default library: ENDF/B-VII.0, `/xsdata/ACE/endfb7/endfb7.xsdata`, suffixes `.03c`-`.18c` = 300-1800 K for every nuclide we need.
- Water S(α,β) for ENDF/B-VII.0: `therm lwtr lwe7.12t` (600 K); `lwtr` is only our label.
- Write steels, zirconium and lead as isotopes, not natural elements, so the same materials work in every library.
- Hexagonal cell: `hexyc` with `set bc 3` (periodic), as the Serpent 1 manual recommends.
- Detectors: `dr <mt> <mat>` only sets the cross section; restrict the region with `dm <mat>`. `-2` is capture, so absorption = `-2` + `-6`; `-7` is ν-fission; `ene <name> 3 ...` is an equal-lethargy grid.
- The `tmp` Doppler entry only broadens upward from the library temperature.

## Physics sanity checks you must apply
- The geometry plot is shown to the group and inspected before the first real run of any new geometry.
- Detector balance Σ(ν-fission)/Σ(absorption) ≈ k_inf within statistics.
- Sodium-cooled cases must have a fast spectrum (no thermal peak; most fissions above ~100 keV).
- Claim a difference only if |Δρ| > 3σ_Δρ.
