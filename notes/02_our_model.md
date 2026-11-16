# Step 2: Our model, one fuel rod

*Draft by Claude, 2026-11-16. Change anything that doesn't sound like us.*

## What we simulate

We simulate one fuel rod and the sodium around it: a **pin cell**.
A real core has thousands of nearly identical rods, so one rod is enough.
At the edge of the cell, a neutron that leaves comes back in on the other side, as if there were identical rods everywhere.
So our model is an infinite reactor with no neutrons escaping, and Serpent gives us $k_\infty$.
A real reactor loses neutrons at its edges, so its $k$ is lower than our $k_\infty$.

Seen from above:

```
 fuel pellet  ->  gap      ->  steel tube   ->  sodium   ->  hexagonal edge
 r = 0.474 cm     (empty)      r = 0.542 cm     between      1.205 cm between
                  r = 0.489 cm                  the rods     rod centres
```

The drawing made by Serpent itself will be added in Step 4.

## The parts

| Part | Material | Size | Temperature | Why it is there |
|---|---|---|---|---|
| Fuel pellet | (U,Pu)O₂ with 16 % plutonium, 9.9 g/cm³ | radius 0.474 cm | 1500 K | This is where fission happens and the heat is made. |
| Gap | empty (helium in reality) | 0.15 mm wide | - | Gives the pellet room to expand. Helium is so thin that neutrons don't notice it. |
| Cladding | steel | 0.5 mm thick | 743 K | Holds the fuel in and keeps radioactive material inside the rod. |
| Coolant | liquid sodium, 0.84 g/cm³ | fills the rest of the hexagon | 743 K | Carries the heat away without slowing the neutrons down much. |

## Where the numbers come from

All numbers come from the OECD/NEA sodium fast reactor benchmark (NEA/NSC/R(2015)9, in `docs/`).
We copied the rod from its large MOX core.
Each number, with its page, is in `decisions_log.md`.

## What we simplified

- We use **fresh fuel**: only uranium, plutonium and oxygen.
- The rod is **infinitely tall**, so no neutrons escape at the top or bottom.
- The gap is **empty**.
