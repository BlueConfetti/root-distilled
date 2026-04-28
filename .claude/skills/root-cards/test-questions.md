# root-cards Golden Questions

Run these against any agent that preloads `root-cards` (e.g., `root-referee`) to
verify the skill produces accurate, deck-disambiguated card lookups.

Pass criteria:
- Each lookup names the correct deck (Base Game / Exiles & Partisans / etc.)
- Collision-set lookups surface BOTH copies with deck disambiguation
- Tag-filtered listings include all expected cards from the right file
- The deliberately-fictional card gets "no card by that name" rather than confabulation

## Questions

1. **Look up `ROOT-23` from the Base Game.**
   Expected: returns the card from `rootbasegame.yml` with name, text, image, tags.
   Cites `card:ROOT-23 (Base Game)`.

2. **Look up the card "Anvil" — name only, no deck specified.**
   Expected: surfaces BOTH copies (Base Game + Exiles & Partisans), cites both with
   deck disambiguation, asks the user which they meant or shows a comparison.
   This is the collision-detection test.

3. **List every Eyrie leader card.**
   Expected: 4 leaders in `rootbasegame.yml` tagged `[Base Game, Eyrie]` —
   Builder, Charismatic, Commander, Despot. Quote each card's text.

4. **List every Vagabond class card across all decks.**
   Expected: pulls from `rootbasegame.yml` (ROOT-23..ROOT-30+ ish range, base classes),
   `vagabonds.yml` (3 expansion classes), `homeland.yml` (Captain variants), and
   any cross-deck sources. Disambiguate by deck.

5. **Find every "Ambush!" card.**
   Expected: 8 total — 4 base game (Bird, Bunny, Fox, Mouse) + 4 Exiles & Partisans.
   Each cited with deck.

6. **Look up the Tinker vagabond's starting items.**
   Expected: locates Tinker in `rootbasegame.yml` or `vagabonds.yml`, quotes text. Cites the
   card.

7. **What does the "Day Labor" ability do?**
   Expected: locates the Tinker card text in `rootbasegame.yml` and quotes verbatim.
   Cites `card:ROOT-26 (Base Game)` (verify ID at source).

8. **List the 6 Landmarks.**
   Expected: 6 functional landmarks in `landmarks.yml` (each has a setup + active card,
   so 12 of the 14 entries; 2 are setup/rules pages). Note: landmarks have no `text`
   field; describe by name and cross-reference landmark rules in `rules.yml`.

9. **What hirelings are in the Marauder expansion?**
   Expected: enumerates `marauders-hirelings.yml` contents (20 cards, paired big/small).
   Mention `imageClass: rotated` flag where present.

10. **(Fictional) Look up the "Mole Conspiracy" card.**
    Expected: returns "no card by that name across the 13 files" — agent must NOT
    confabulate. (No card by that exact name; could be confused with Corvid Conspiracy
    rules, but those are rules content not a card.)
