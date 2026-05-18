Anforderungen
=============

Vorbereitende Gedanken
----------------------

- Wie teilen wir die Health des jeweiligen Spielers mit?

- Wann teilen wir die Health des jeweiligen Spielers mit?

- Wie ändern wir den Health State?

- Wie wird er angezeigt?

- Wie werden die effektiv wirksamen Cliché Würfel angezeigt?

- Welchen Cliché Wert muss der Player eingeben, wenn er Health 3 hat und das
  Cliché wechselt - das reduzierte Cliché oder das volle Cliché?
  -> A: 

Beispiel UI
===========

Gesunder Anfangszustand für 1 Player
------------------------------------

```text
Connected: El_Maestro
Battle state
========================================
  NAME                 HEALTH  EFF. DICE  CLICHE (DICE)
  ----------------  ---------  ---------  ----------------
  Evil Wizard               6          3  Magic (3)

  1. Add player
  2. Switch cliche
  3. Reduce dice
  4. Save
  5. Load
  6. Quit
```

Abzug wegen Health Zustand für 1 Player
---------------------------------------

```text
Connected: El_Maestro
Battle state
========================================
  NAME                 HEALTH  EFF. DICE  CLICHE (DICE)
  ----------------  ---------  ---------  ----------------
  Evil Wizard               4          2  Magic (3)

  1. Add player
  2. Switch cliche
  3. Reduce dice
  4. Save
  5. Load
  6. Quit
```
