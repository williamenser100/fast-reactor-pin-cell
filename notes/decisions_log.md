# Decisions log

Every modelling number and choice gets one row: value, unit, source, reason.
This table feeds the Methods section of the report.

| Date | Quantity | Value | Unit | Source | Reason | Who |
|---|---|---|---|---|---|---|
| 2026-11-16 | Reference design | MOX-3600 core | - | NEA/NSC/R(2015)9 p. 11-16 | Only large SFR core with MOX fuel in the benchmark | Claude, agreed by group |
| 2026-11-16 | Fuel pellet outer radius | 0.4742 | cm | Table 2.4, p. 14 | Hot dimensions; solid pellet for simplicity (benchmark volume fraction hints at a small central hole) | Claude, agreed by group |
| 2026-11-16 | Gap outer radius (cladding inner) | 0.4893 | cm | Table 2.4, p. 14 | Hot dimensions | Claude, agreed by group |
| 2026-11-16 | Gap content | void | - | p. 15 (helium bond) | Helium is too thin to affect neutrons | Claude, agreed by group |
| 2026-11-16 | Cladding outer radius | 0.5419 | cm | Table 2.4, p. 14 | Includes smeared wire wrap | Claude, agreed by group |
| 2026-11-16 | Pin pitch | 1.2053 (P/D 1.112) | cm | Derived from Table 2.4, p. 14 | Area inside the wrapper tube (flat-to-flat 20.7468 - 2 x 0.4525 cm) divided by 271 pins | Claude, agreed by group |
| 2026-11-16 | Fuel | fresh (U,Pu)O2, O/M 2.00 | - | Our choice | No fission products or minor actinides; Pu content becomes one clean parameter | Claude, agreed by group |
| 2026-11-16 | Pu content | 16 | wt% of heavy metal | Table 2.11, p. 19 | Inner-core BOC value 15.9 %, rounded | Claude, agreed by group |
| 2026-11-16 | Pu vector 238/239/240/241/242 | 2.6/52.9/29.1/6.1/9.3 | wt% | Table 2.11, p. 19 | Inner-core axial average, normalised to 100 | Claude, agreed by group |
| 2026-11-16 | 235U in uranium | 0.17 | wt% | Table 2.11, p. 19 | Depleted uranium | Claude, agreed by group |
| 2026-11-16 | Fuel density | 9.94 | g/cm3 | Derived from Table 2.11, p. 19 | Same heavy-metal atom density as benchmark (0.02215 /b-cm); 90 % of MOX theoretical density | Claude, agreed by group |
| 2026-11-16 | Cladding material | ODS steel (Fe, Cr, Al, ...) | atoms/b-cm | Table 2.13, p. 20 | Copied exactly, written as isotopes | Claude, agreed by group |
| 2026-11-16 | Cladding density | 7.13 | g/cm3 | Derived from Table 2.13, p. 20 | Sum of the given number densities | Claude, agreed by group |
| 2026-11-16 | Sodium density | 0.837 (2.1924E-02 atoms/b-cm) | g/cm3 | Table 2.13, p. 20 | Liquid sodium at about 470 °C | Claude, agreed by group |
| 2026-11-16 | Fuel temperature | 1500 | K | Table 2.1, p. 11 (1227 °C) | Matches the .15c library exactly | Claude, agreed by group |
| 2026-11-16 | Cladding and coolant temperature | 743 | K | Table 2.1, p. 11 (470 °C) | .06c library broadened up with tmp | Claude, agreed by group |
| 2026-11-16 | Axial extent | infinite (2D) | - | Our choice | Pin-cell study, no axial leakage | Claude, agreed by group |
| 2026-11-16 | Cell boundary | hexagonal, periodic (hexyc, set bc 3) | - | Serpent 1 manual | Equivalent to reflective for a symmetric cell | Claude, agreed by group |
| 2026-11-16 | Nuclear data | ENDF/B-VII.0 | - | notes/cluster_inventory.md | Cluster default; all isotopes at all temperatures | Claude, agreed by group |
