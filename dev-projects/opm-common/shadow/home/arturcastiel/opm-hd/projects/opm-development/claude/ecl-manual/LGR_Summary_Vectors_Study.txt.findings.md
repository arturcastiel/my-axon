# Shadow: ecl-manual/LGR_Summary_Vectors_Study.txt
project: opm-common
source: Hand-curated study guide for Eclipse 2024.1 LGR manuals
indexed: 2026-05-05

## Summary
Study guide cross-referencing LGR summary vector topics across 4 Eclipse manuals.
Key manual locations for SMSPEC LGR arrays and LGR file structure.

## SMSPEC LGR Arrays (FileFormatsReferenceManual.pdf pp. 8-11)

| Array | Definition |
|---|---|
| LGRS | LGR name associated with each summary vector |
| NUMLX | I-position of block/completion within the local grid |
| NUMLY | J-position of block/completion within the local grid |
| NUMLZ | K-position of block/completion within the local grid |
| LGRNAMES | Names of all local grids defined in the run |
| LGRVEC | Number of summary vectors associated with each LGR |
| LGRTIMES | Total number of local ministeps for each LGR |

## ROOT.LGR File Structure (FileFormatsReferenceManual.pdf pp. 11-12)
- Contains only LB*, LC*, LW* vectors
- **Also contains TIME and YEARS for local ministeps** → explains why LGRVEC = 2 + N_lgr
- Always unified (single file, not split per report step)
- Table 3.3 record sequence: SEQHDR → LGRNAME → MINISTEP → PARAMS

## Vector Naming Convention (FileFormatsReferenceManual.pdf pp. 12-13, Table 3.4)
| Prefix | Category | Additional spec data required |
|---|---|---|
| LB | Local Block | LGR name + local I,J,K (LGRS + NUMLX/Y/Z) |
| LC | Local Connection | LGR name + well name + local I,J,K (LGRS + WGNAMES + NUMLX/Y/Z) |
| LW | Local Well | LGR name + well name (LGRS + WGNAMES; NUMLX/Y/Z = 0,0,0) |

## Key Facts for OPM Implementation
- LGRVEC[i] = "number of summary vectors associated with LGR i" = **2 + count(LGR-i vectors)**
  The +2 accounts for TIME and YEARS which are always written to the ROOT.LGR file per LGR.
- LGRTIMES[i] = total local ministeps for LGR i (dynamic — increases each simulation step)
- NOSUMLGR keyword suppresses ROOT.LGR creation; does NOT suppress LGRS/NUMLX etc. in SMSPEC

## Manual Index
| Topic | Location |
|---|---|
| SMSPEC LGR arrays (LGRS, NUMLX, LGRNAMES, etc.) | FileFormatsReferenceManual.pdf pp. 8-11 |
| ROOT.LGR file structure | FileFormatsReferenceManual.pdf pp. 11-12 |
| LW*/LC*/LB* vector naming | FileFormatsReferenceManual.pdf pp. 12-13 |
| SUMMARY section LW* usage | EclipseReferenceManual.pdf p. 127 |
| SUMMARY section LC* usage | EclipseReferenceManual.pdf p. 126 |
| SUMMARY section LB* usage | EclipseReferenceManual.pdf pp. 125-126 |
| NOSUMLGR keyword | EclipseReferenceManual.pdf p. 188 |
| LGR summary output theory | EclipseTechnicalDescription.pdf pp. 66-68 |

## Findings Log
| date | context | finding |
|------|---------|---------|
| 2026-05-05 | PR02 reference analysis | LGRVEC = 2 + N_lgr confirmed: "also contains TIME and YEARS for local ministeps" |
| 2026-05-05 | PR02 reference analysis | LGRTIMES = total local ministeps per LGR — dynamic, per-LGR |
