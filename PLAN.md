# Our project plan: what changes k in a fast reactor?

*Simple version, written 2026-11-11.
The old, very detailed plan is kept in `notes/old_detailed_plan.md` in case we need a technical detail from it.*

## The project in five sentences

1. A nuclear reactor runs on a chain reaction: neutrons split heavy atoms, and each split releases new neutrons.
2. The number $k$ tells us whether that chain reaction grows ($k > 1$), stays steady ($k = 1$) or dies out ($k < 1$).
3. We build a computer model of **one fuel rod from a fast reactor** and let the program Serpent calculate $k$ for it.
4. Then we change one thing at a time (fuel, coolant, spacing and so on) and see how $k$ changes.
5. The goal is to **understand why** each change makes $k$ go up or down, not just to produce numbers.

## How we work

- **We think, Claude codes.** Claude Code writes all the code and runs the cluster. Our job is to understand the physics, make predictions and explain the results.
- **Never accept something you cannot explain.** If Claude writes or says something you don't follow, ask: *"Explain that as if I have never seen it before."*
- **Every step ends with a short note in our own words** in the `notes/` folder. These notes become the report, so writing them is not extra work.
- **Predict before we simulate.** Writing down what we *expect* before we see the result is the best way to learn. A wrong prediction is fine; finding out *why* it was wrong is the interesting part.

### The little Linux you need

You do not need to learn Linux. Claude handles the cluster.
If you want to look around yourself, these commands are safe (they only *look*, they change nothing):

| Command | What it does |
|---|---|
| `ssh megaraptor` | Log in to the cluster (from the laptop terminal) |
| `ls` | List the files in the folder you are in |
| `cd foldername` | Go into a folder (`cd ..` goes back up, `cd` alone goes home) |
| `cat filename` | Print a text file on the screen |
| `exit` | Log out again |

### Three cluster facts everyone must know

1. **We all share one login (`mp10`).** Only *one* test run may be running on the login computer at a time, for the whole group. Say in the group chat before you start anything.
2. **The account and all files on it are deleted on 15 December 2026.** Everything important is copied to our laptops and git well before that.
3. **Don't run Serpent on the cluster yourself** without checking with whoever is driving Claude Code. Breaking the cluster rules can get the group blocked. The full rules are in `CLAUDE.md`, and Claude follows them.

---

## Step 1: What is a fast reactor?

**Goal:** everyone can explain, in plain words, what a reactor does, what $k$ means, and what makes a reactor "fast".

### A starting explanation

**Fission.** A heavy nucleus such as uranium-235 or plutonium-239 can absorb a neutron and split in two. This releases energy (heat) and 2 to 3 new neutrons.

**The chain reaction and k.** Those new neutrons can cause new fissions, which release more neutrons, and so on. Follow one "generation" of neutrons to the next:

$$k = \frac{\text{number of neutrons in the next generation}}{\text{number of neutrons in this generation}}.$$

A neutron can meet one of three fates: it causes a fission (good, makes new neutrons), it gets absorbed without fission (lost), or it escapes out of the reactor (lost).
So $k$ is a competition between **making** neutrons and **losing** them.

**Slow versus fast neutrons.** Neutrons are born very fast (about 20 000 km/s).
- In an ordinary reactor (the kind in Sweden), **water slows the neutrons down** to "thermal" speed (about 2 km/s). Slow neutrons are much more likely to cause fission in uranium-235, so a few percent of uranium-235 is enough fuel. This is a **thermal reactor**.
- In a **fast reactor** there is **nothing to slow the neutrons down**. The neutrons stay fast. Fast neutrons cause fission less easily, so the fuel needs more fissile material: typically 15 to 25 % plutonium mixed with uranium ("MOX" fuel).

**Why build a fast reactor at all?** When plutonium-239 fissions with *fast* neutrons, it releases more spare neutrons. Those spare neutrons can turn the uranium-238 in the fuel (which is otherwise mostly useless) into new plutonium-239. This is called **breeding**: the reactor can make new fuel while it runs, and it can also burn long-lived nuclear waste.

**Consequences for the design.**
- The coolant must not slow neutrons down, so no water. Liquid **sodium** is the most common choice (lead and helium are alternatives).
- The fuel rods are packed **tightly** in a **hexagonal** pattern, so there is little coolant between them.

### What to do

1. Read the explanation above. Ask Claude about anything unclear (*"Why does water slow neutrons down but sodium doesn't?"*).
2. Each person answers the check questions below on their own, then compare as a group.
3. Write a 1-page note together: `notes/01_what_is_a_fast_reactor.md`.

### You understand Step 1 when you can answer

- What is $k$, and what happens to the reactor if $k$ is 0.98, 1.00 or 1.02?
- What are the three things that can happen to a neutron?
- What does a moderator do, and why does a fast reactor not have one?
- Why does a fast reactor need much more plutonium or uranium-235 than a thermal reactor?
- What is breeding, and why does it need fast neutrons?

> **Prompt for Claude:** *"We are doing Step 1 of PLAN.md. Quiz us with the check questions one at a time, correct our answers, and then help us write notes/01_what_is_a_fast_reactor.md in our own words."*

---

## Step 2: Our model, one fuel rod

**Goal:** know exactly what we are simulating, and why each part is there.

### Why only one fuel rod?

A real reactor core has tens of thousands of fuel rods, all nearly identical.
We simulate **one rod plus the coolant around it**, called a **pin cell**.
The edge of the cell acts like a **mirror**: a neutron that hits it bounces back in, exactly as if there were identical neighbours everywhere.
That makes our model an **infinitely large reactor** with no neutrons escaping.
The $k$ for such an infinite reactor is called **$k_\infty$** ("k-infinity").
A real reactor loses neutrons out of its edges, so its $k$ is lower than our $k_\infty$. We must remember this when we interpret results.

### What is in the pin cell

Seen from above, from the centre outwards:

```
 fuel pellet   ->  thin gap   ->  steel cladding tube  ->  liquid sodium  ->  hexagonal edge (the "mirror")
 (U,Pu) oxide      (empty)        holds the fuel in       carries heat away
 about 1 cm wide                  about 0.5 mm thick
```

### What to do

1. Get realistic numbers for each part: sizes, densities, plutonium content, temperatures.
   The best source is the OECD/NEA sodium fast reactor benchmark (NEA/NSC/R(2015)9). It is a free PDF from oecd-nea.org; put it in `docs/`.
   Claude reads it and proposes the numbers.
2. For **every number**, make sure the group knows what it means and why that value was chosen. Each one gets a line in `notes/decisions_log.md`.
3. Claude makes a drawing of the pin cell from the actual model (Serpent can draw the geometry).
4. Write `notes/02_our_model.md`: the drawing, a table of the parts, and one sentence per part saying why it matters.

### You understand Step 2 when you can answer

- What is a pin cell, and why is it enough to simulate one?
- What does the mirror boundary do, and why is our $k_\infty$ higher than the $k$ of a real reactor?
- What is each layer made of, and what is its job?
- Where did our numbers come from?

> **Prompt for Claude:** *"We are doing Step 2 of PLAN.md. Read the NEA benchmark in docs/, propose the baseline numbers for our pin cell with page references, explain each one in plain language, and log them in notes/decisions_log.md. Then help us write notes/02_our_model.md."*

---

## Step 3: What can change k? Make predictions

**Goal:** list the factors that could change $k_\infty$, predict what each one does and why, and pick the 3 to 4 factors we will actually study.

### The key idea: three levers

With no neutrons escaping, $k_\infty$ is simply

$$k_\infty = \frac{\text{neutrons produced by fission}}{\text{neutrons absorbed}}.$$

So any change can only move $k_\infty$ in three ways:

1. **Production lever:** more or fewer fissions (for example more or less fissile material).
2. **Absorption lever:** more or fewer neutrons wasted by being absorbed in things that don't fission (coolant, cladding, oxygen and so on).
3. **Speed lever:** the neutrons become faster or slower on average, and that changes how likely each reaction is.

Every result we find should be explained with one or more of these levers.

### Candidate factors

Fill in the last two columns **before** running anything:

| # | Factor | What we would vary | Our prediction: k up or down? | Why? (which lever) |
|---|---|---|---|---|
| 1 | Plutonium content | 10 % to 30 % of the heavy metal | | |
| 2 | Pin spacing | rods closer together or further apart (more or less coolant) | | |
| 3 | Losing the coolant | sodium density from 100 % down to 0 % (boiling or a leak) | | |
| 4 | Coolant type | sodium, lead, helium, and **water** as a contrast | | |
| 5 | Fuel temperature | 300 K to 1800 K | | |
| 6 | Cladding material | different steels, or zirconium as in thermal reactors | | |
| 7 | Fuel type | oxide, nitride, metal | | |

### Choosing what to study

Pick **3 or 4** factors. Doing a few well, with a clear explanation, is worth more than doing many.
Good choices for beginners have an effect that is large and easy to explain.
A suggested set: **1 (plutonium content)**, **4 (coolant type, including water)** and **3 (losing the coolant)**, plus **2 or 5** if time allows.
Factor 4 with water is also our proof that the sodium model really is "fast": the water case should look completely different.
Check the choice with the supervisor.

### What to do

1. Discuss each factor as a group. Ask Claude for hints, not answers, if you get stuck.
2. Fill in the table in `notes/03_factors_and_predictions.md`.
3. Write down which factors we chose and why.

### You understand Step 3 when you can answer

- What are the three levers, and can you give an example of each?
- For each chosen factor: what do we expect, and which lever is behind it?

> **Prompt for Claude:** *"We are doing Step 3 of PLAN.md. Here are our predictions for each factor: [...]. Don't tell us the right answers yet. Ask us questions that test our reasoning, then help us write notes/03_factors_and_predictions.md."*

---

## Step 4: Our first simulation

**Goal:** run the baseline pin cell once, get $k_\infty$ with its uncertainty, and show that the neutrons really are fast.

### How Serpent works, in short

Serpent is a **Monte Carlo** program: it plays a game of chance with millions of simulated neutrons.
For each neutron it rolls dice to decide how far it flies, what it hits and what happens there, using measured probabilities called **cross sections** (the "nuclear data").
It runs generation after generation (called **cycles**).
The first cycles are thrown away while the neutrons spread out into their natural pattern; the rest are averaged.
The result is $k_\infty$ **with an error bar** (written $\pm\sigma$), because it comes from random sampling.
More neutrons give a smaller error bar, but slowly: **4 times more neutrons only halves the error**.

### What happens in this step

1. Claude writes the Serpent input file and explains **every line** to us.
2. Claude draws the geometry. **We look at the picture together** and check that it matches our drawing from Step 2.
3. Claude does a short test run to measure how much time and memory a real run needs, then sends the real run to the cluster.
4. We get $k_\infty \pm \sigma$ for the baseline.
5. Claude also runs a **water-cooled** pin cell and plots the **neutron energy spectrum** of both (how many neutrons there are at each speed). The water case should have a large bump of slow neutrons; the sodium case should not. This is our proof that the model is a fast reactor.
6. Write `notes/04_first_results.md`: the picture, our $k_\infty$, the spectrum plot, and what each one tells us.

### You understand Step 4 when you can answer

- What does Serpent actually do with each neutron?
- What do the "cycles" mean, and why are the first ones thrown away?
- Why does $k_\infty$ have an error bar, and how do we make it smaller?
- What in the spectrum plot shows that our sodium pin cell is fast?

> **Prompt for Claude:** *"We are doing Step 4 of PLAN.md. Write the baseline Serpent input and explain every line as if to someone who has never seen Serpent. Show us the geometry plot and wait for our OK before running anything. Follow CLAUDE.md strictly on the cluster."*

---

## Step 5: Change one thing at a time

**Goal:** run the factors we chose in Step 3, one at a time, with everything else kept at the baseline.

### What happens in this step

1. Claude turns the input from Step 4 into a small program that can produce a variation automatically (for example "the same pin cell, but with 20 % plutonium").
2. Claude first checks that the program reproduces the Step 4 result exactly.
3. For each chosen factor, Claude runs all its values on the cluster and makes a plot of $k_\infty$ against that factor, with error bars.
4. We compare each plot with our prediction from Step 3.

### When is a difference real?

Two results always differ a little just because of randomness.
We only call a difference **real** if it is bigger than about **three times its error bar**.
Claude calculates this for us; we just need to understand why it matters.

### You understand Step 5 when you can answer

- Why do we change only one thing at a time?
- For each plot: does $k_\infty$ go up or down, and was our prediction right?
- Which differences are real, and which could be just randomness?

> **Prompt for Claude:** *"We are doing Step 5 of PLAN.md for factor [X]. Show us the values you plan to run and the job size before submitting. When the results are in, plot k_inf against [X] with error bars and compare with our prediction in notes/03."*

---

## Step 6: Explain and write

**Goal:** for every factor, explain *why* $k_\infty$ changed, using the three levers.

### What happens in this step

1. For each factor, Claude can split $k_\infty$ into its parts: how many neutrons were produced, and where neutrons were absorbed (fuel, cladding or coolant), plus how the spectrum changed. This tells us *which lever* was responsible.
2. We write one paragraph per factor: what we predicted, what happened, and why.
3. Where our prediction was wrong, we explain what we misunderstood. These are often the most interesting parts of the report.
4. A final summary figure ranks the factors from biggest to smallest effect on $k_\infty$.
5. Write the report from our notes. Structure: introduction (Step 1), method (Steps 2 and 4), results (Step 5), discussion (Step 6), conclusions.
   Follow the writing guide `Stylistic_Principles.pdf`; Claude can check the text against it.

### Remember in the discussion

Our model is an **infinite** reactor with no neutrons escaping.
For some factors, especially losing the coolant, a real reactor can behave differently because more neutrons escape.
We must say this clearly instead of claiming our numbers describe a real reactor.

> **Prompt for Claude:** *"We are doing Step 6 of PLAN.md. For factor [X], show us the production, absorption and spectrum results, ask us which lever explains the change, and then help us write the paragraph."*

---

## Rough timeline

The cluster account is deleted on **15 December 2026**, so all simulations must be finished and downloaded by about **8 December**.

| Week | Dates | Steps |
|---|---|---|
| 1 | 11-17 Nov | Steps 1, 2 and 3 (understanding, model, predictions) |
| 2 | 18-24 Nov | Step 4 (first simulation) |
| 3 | 25 Nov - 1 Dec | Step 5 (the factor runs) |
| 4 | 2-8 Dec | Step 6 (explaining), last runs and download everything |
| after | | Write the report (fill in our deadline: ____ ) |

## Who does what

Everyone does Steps 1 and 3 together, since they are about understanding.
After that, pair up so at least two people understand each part:

| Role | Owns |
|---|---|
| Claude Code driver (2 people) | Talks to Claude, runs the cluster work, keeps git updated |
| Physics (1-2 people) | Numbers for the model, `decisions_log.md`, sources |
| Report (everyone, one editor) | Turns the `notes/` into the report |

## Word list

| Word | Meaning |
|---|---|
| Fission | A heavy nucleus splits, releasing energy and 2-3 neutrons |
| $k$, $k_\infty$ | Neutrons in the next generation divided by neutrons in this one; $k_\infty$ is the same for an infinite reactor with no escape |
| Fast / thermal neutron | Neutron at high speed (just born) / slowed down to the speed of the surrounding atoms |
| Moderator | Material that slows neutrons down (water in normal reactors) |
| MOX | "Mixed oxide" fuel: uranium and plutonium oxide |
| Breeding | Turning uranium-238 into new plutonium-239 fuel using spare neutrons |
| Pin cell | One fuel rod plus its share of coolant |
| Cladding | The metal tube that holds the fuel |
| Coolant | The liquid or gas that carries heat away (sodium in our case) |
| Cross section | The probability of a neutron reacting with a nucleus (depends on neutron speed) |
| Spectrum | How many neutrons there are at each speed (energy) |
| Monte Carlo | Calculating something by simulating many random events |
| Cycle | One simulated neutron generation in Serpent |
| Error bar, $\sigma$ | How uncertain a Monte Carlo result is because of randomness |
| Cluster | A group of powerful computers we log in to remotely |
| Job | A calculation we send to the cluster to run when there is room |
