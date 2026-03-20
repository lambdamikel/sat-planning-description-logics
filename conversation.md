# Development Conversation

*Transcript of the collaborative session between Michael Wessel and Claude (Anthropic), March 2026.*
*This conversation documents the iterative development of role-based DL encodings for the Wolf-Goat-Cabbage SAT planning problem.*

---

## Initial Task

**Michael:** Refactor the wkz-via-porter.racer file, which is a classic sat planning examples (wolf-cabbage-goat) in KRSS description logic syntax. Study the slide in PDF given in this folder. Then, create a wkz-via-porter2.racer file in which you create a more compact representation that uses DL roles / relations instead of the proliferated symbols for riverbank a, b. Instead of "goat-a" have something like "(and goat (some location a))". Obviously, the axioms need to be rewritten. For the actions such as "wolf-boat-from-a-to-b" also find a way to represent the fact that the wolf is in the boat going from a to b. Devise a compact representation.

**Claude:** *[Studies the original .racer file and the PDF slides describing the WGC riddle, its SAT planning reduction, and the KRSS encoding]*

The key insight: instead of proliferated symbols like `goat-a`, `goat-b`, `goat-boat-from-a-to-b`, `goat-boat-from-b-to-a` (4 symbols per object, 20 total), we use **5 functional roles** (features): `goat-loc`, `wolf-loc`, `cabbage-loc`, `ferryman-loc`, `boat-loc` and **4 shared location concepts**: `a`, `b`, `from-a-to-b`, `from-b-to-a`.

Representation mapping:

| Original (atomic) | New (role-based) |
|---|---|
| `goat-a` | `(some goat-loc a)` |
| `goat-b` | `(some goat-loc b)` |
| `goat-boat-from-a-to-b` | `(some goat-loc from-a-to-b)` |
| `wolf-boat-from-a-to-b` | `(some wolf-loc from-a-to-b)` |

*[Creates `wkz-via-porter2.racer` with object-centric location roles, all declared as features]*

---

## Performance Problem: Too Slow

**Michael:** Now it takes a long time to compute. Is there a way to speed up the sat test by changing it a bit?

**Claude:** The slowdown comes from the tableau reasoner having to deal with many complex `(some role concept)` expressions everywhere. Each one forces the reasoner to create/manage role fillers, and they can't be "absorbed" into the concept hierarchy like atomic names can (GCI absorption).

The fix: **define named concepts as abbreviations** for all `(some obj-loc place)` expressions, then use those names in axioms. This preserves the role-based semantics but gives the reasoner atomic-like names to optimize with.

*[Creates version with `(define-concept goat-at-a (some goat-loc a))` etc., 20 defined concepts]*

---

## Symbol Proliferation Returns

**Michael:** Yes, but now we have proliferated symbols again...

**Claude:** You're right -- the `define-concept` abbreviations just reintroduced the symbol proliferation we were trying to eliminate. The whole point was to avoid 20 separate object-location names.

The fundamental tension is: Racer's tableau is fast with atomic concept names on the LHS of GCIs, but slow with complex `(some role concept)` expressions everywhere.

*[Creates version keeping compact 9-symbol vocabulary, adds `:domain`/`:range` constraints and more ABox hints]*

---

## Still Slow, Features Expensive

**Michael:** Still slow. The problems are the `(implies (some ...) ...)` axioms. Do these roles need to be features? Can we get a version without features? Features are expensive, too.

**Claude:** *[Drops `:feature t` from all location roles, replaces with `(at-most 1 role)` in the state axiom]*

---

## At-Most Also Expensive

**Michael:** Still slow. Are the "at-most 1" restrictions on the roles really required?

**Claude:** No, they're not. The axiom structure naturally produces at most one filler per role per state:

1. Start creates one filler per role
2. No-teleportation propagates exactly one `(some ...)` to the next state via `(all next ...)`
3. The state axiom's `(or ...)` is already satisfied by that propagated filler -- no second filler gets created

The tableau is lazy -- it only creates fillers when forced by existential restrictions. No axiom ever forces a second filler for the same role.

*[Removes all `(at-most 1 ...)` restrictions]*

---

## Location-Centric Roles

**Michael:** Would it be easier if we added a "at" role for the locations a, b, boat-from-a-to-b, boat-from-b-to-a, and "at" would refer to the objects "at" that location? So instead of the objects keeping track of their locations, the locations are keeping track of the objects "at" them...

**Claude:** Interesting idea -- inverting the roles so locations track objects instead of objects tracking locations.

**Symbol count:** 4 roles (`at-a`, `at-b`, `at-ab`, `at-ba`) + 5 object concepts (`goat`, `wolf`, `cabbage`, `ferryman`, `boat`) = still 9 symbols.

**Key difference:** axioms like safety and capacity now use the **same role** throughout instead of mixing 3-5 different roles:
- `(implies (and (some at-a wolf) (some at-a goat)) (some at-a ferryman))` -- all `at-a`

*[Creates location-centric version]*

---

## Missing State Labels

**Michael:** OK, but for reading of the labels, we now indeed need atomic concepts again it seems... so introduce labels for the states. Something like `(implies (some ...) <label>)`.

**Claude:** Right -- the query `(retrieve (?x (types ?x)) (?x state))` needs atomic labels to show readable results. These are cheap "tagging" axioms: `(some ...)` on the LHS but just an atomic name on the RHS (no new fillers created).

*[Adds 20 labeling axioms like `(implies (some at-a goat) goat-a)`]*

---

## State Propagation Bug

**Michael:** OK, there is a bug. Only s1 and s15 are "state" instances.

**Claude:** *[Proposes adding explicit `(instance sN state)` declarations]*

**Michael:** Instead, declare the "next" features as having domain and range "state".

**Claude:** *[Changes to `(define-primitive-role next :feature t :domain state :range state)`]*

Now any individual that appears as domain or range of a `next` edge is automatically classified as `state`.

---

## Open-World Semantics Issue

**Michael:** Now, there is another thinking error: due to open domain semantics, `(implies state .. (some next ...))` does NOT necessarily apply to the explicitly modeled "next" state successors. Rather, you would need an `(all next ...)` somewhere.

**Claude:** You're right. `(some next ...)` can be satisfied by a fresh anonymous individual rather than the one linked by `(related ...)`. With `(all next ...)`, the constraint applies to ALL next-fillers, which includes the explicitly asserted one.

*[Changes state axiom: `(some next state)` to `(some next top)`, wraps object persistence disjunctions in `(all next ...)`]*

---

## The Uniqueness / Soundness Bug

**Michael:** Still not correct:
```
((?x s2)
  ((... ?x)
   ((*top* top)
    (boat-from-a-to-b)
    (ferryman-boat-from-a-to-b)
    (state))))
```

**Claude:** The real issue isn't `some` vs `all next` -- it's deeper. Without uniqueness enforcement, the safety constraints are **toothless**. Here's why:

In s2, if goat stays at-a and wolf stays at-a, safety requires `(some at-a ferryman)`. Without uniqueness, the reasoner simply creates an **additional** at-a filler of type `ferryman` -- even though ferryman is already at-ab (on the boat). The ferryman is at two places at once, and nothing prevents it.

In the original encoding, `(disjoint ferryman-a ferryman-b ferryman-boat-from-a-to-b ferryman-boat-from-b-to-a)` prevented this.

With location-centric roles (`at-a`, `at-b`, ...), we can't use features for per-object uniqueness because multiple objects legitimately share a location. We need explicit exclusion axioms per object across locations:

```
(implies (some at-a goat) (and (not (some at-b goat))
                               (not (some at-ab goat))
                               (not (some at-ba goat))))
```

That's 5 objects x 4 locations = 20 axioms.

---

## Adding Exclusion Axioms

**Michael:** Let's try adding these.

**Claude:** *[Adds 20 per-object location exclusion axioms]*

**Michael:** Excellent! And still reasonably fast.

---

## Paper and GitHub

**Michael:** Now, can you create a LaTeX paper that described our journey? Start with my original model. Have a sub-section for each approach we tried. Describe the challenges. Use mathematical DL notation, and Racer notation as well.

**Claude:** *[Creates `paper.tex` with full mathematical DL notation and KRSS syntax, covering all approaches, fixes LaTeX compilation issues, produces 7-page paper]*

**Michael:** Can you also create a GitHub project for this - "sat-planning-description-logics". Add a README.MD containing the paper descriptions. Create a src/ directory that contains the different versions we tried.

**Claude:** *[Creates GitHub repo at https://github.com/lambdamikel/sat-planning-description-logics with all 9 encoding variants numbered 00-08, paper, and comprehensive README]*

---

## Summary of Approaches

| # | Approach | Symbols | Result |
|---|----------|---------|--------|
| 00 | Original (atomic concepts) | 20+ | Fast, correct (baseline) |
| 01 | Object-centric roles + features | 9 | Too slow (GCIs + features) |
| 02 | Defined concept abbreviations | 9+20 | Symbol proliferation returns |
| 03 | Features + domain/range + hints | 9 | Still too slow |
| 04 | Nominals/fills approach | 9+4 | Racer doesn't support `fills` |
| 05 | No features, with at-most | 9 | Still too slow |
| 06 | No features, no at-most | 9 | Correctness concerns |
| 07 | Location-centric roles | 9 | Unsound (no uniqueness) |
| 08 | Location-centric + exclusion | 9+20 labels | Correct, reasonably fast |
