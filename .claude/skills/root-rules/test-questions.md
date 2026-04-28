# root-rules Golden Questions

Run these against `root-referee` (or directly in a session that has `root-rules`
preloaded) to verify the skill produces cited, faithful answers.

Pass criteria:
- Every claim cites `rule:X.Y.Z` or `faq:N` (backticks).
- The deliberately-out-of-scope question gets "Law is silent" or equivalent.
- Conflicts surface the Golden Rule of Precedence with a citation.

## Questions

1. **What is the precedence order when a card conflicts with the Law?**
   Expected: cites `rule:1.1.X` (Rules Conflicts → Precedence). Card overrides Law;
   Law overrides Learning to Play; faction/hireling rule overrides general rule
   when both cannot be followed simultaneously.

2. **Is the term "cannot" overridable by another card or effect?**
   Expected: "no — `cannot` is absolute" with a cite from Golden Rules → Use of Cannot.

3. **What three phases make up a player's turn?**
   Expected: Birdsong, Daylight, Evening. Cite Game Structure → Turn Structure.

4. **In the Eyrie Dynasties faction, what triggers Turmoil?**
   Expected: cite within rule 7 (Eyrie). Walk to Forced Order → Turmoil.

5. **Can I interrupt another player's compound action (e.g., Marquise's March) with an effect?**
   Expected: no, unless explicitly allowed. Cite Golden Rules → Interrupts.

6. **When pieces are removed simultaneously, in what order are triggered effects resolved?**
   Expected: all pieces removed first, then effects trigger. Cite the Pieces section.

7. **How is a card from the Riverfolk Company's hand visibility different from a normal player's hand?**
   Expected: cites `faction:riverfolk$11.2.3` (the hand-visibility exception in the Riverfolk section).

8. **Look up FAQ entries about ambush cards.**
   Expected: enumerates matching FAQ entries with `faq:N` cites. Uses `grep` on faq.yml.

9. **(Out of scope) What does the Frogs faction do during their Daylight phase?**
   Expected: "Law is silent on a Frogs faction" or "no faction by that name in the
   Oct 2025 Law" — agent must NOT invent. (The Frogs are not a published Root faction.)

10. **What happens at end of Evening that is preserved across the turn boundary?**
    Expected: hireling effects, end-of-Evening triggers per `rule:22.` Cite that node.
