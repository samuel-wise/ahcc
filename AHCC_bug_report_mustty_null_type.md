# AHCC 6.0 (68000): compiler-internal bus error in mustty() on a constant-folded '||' operand

Status: DRAFT bug report - NOT YET SUBMITTED. English first, Nederlandse versie below.

## Summary

AHCC 6.0 for the 68000 crashes with an internal bus error while type-checking a
logical-OR expression of the form `(constant || global)` when that expression is
used as an argument to a prototyped function call. The compiler dies without
printing any C diagnostic. It is 100% reproducible and independent of the
emulator/machine (observed under Hatari with exception debugging armed; AHCC's
own crash handler prints `Crash at text+00015f42`).

Root cause analysis below points at the constant-folding guard in
`binary_types()` (E2.C): for `&&`/`||` nodes with an `ICON` operand it calls
`ycanon(np)` and `return`s, bypassing the only statement that ever sets the
node's `->type` (`np->type = CC_type(...)` later in the same function). When
such a never-typed node later reaches `mustty()` (MD.C), the first statement
`short tok = np->type->token;` dereferences a NULL `->type` and bus-faults.

## Affected version / build

- AHCC 6.0, 68000 binary distribution (AHCCST.BIN; compiler body AHCCST_P.TTP,
  released 2020-11-27). Crash PC = text+0x15F40 (AHCC prints text+00015f42 with
  its +2 file-offset convention).
- Tested through the AHCC IDE/autobuild flow (Pexec of AHCCST_P.TTP) under
  Hatari (exception debugging enabled so the CPU stops at the fault).
  Expected to crash on real hardware too - it is a plain read from address $10
  (NULL+0x10) in user mode.
- AHCC 5.6: NOT yet tested.

## Minimal reproducer (self-contained, no libraries)

File MIN48.C - compile it as a single-file project; AHCC dies with no
diagnostic printed:

```c
static const char g_10 = 0x06;
static void func_1(void);
static unsigned char func_2(char q);

static void func_1(void)
{
    char l_9 = 0x07;
    func_2((l_9 != (3 || g_10)));
}

static unsigned char func_2(char q)
{
    return 0;
}

int main (void)
{
    func_1();
    return 0;
}
```

Observed at the fault (CPU stopped by the emulator debugger):
bus error READ of address $00000010; A0 = 0; PC at AHCCST_P.TTP text+0x15F40
(mustty+14); the faulting instruction pair is
`movea.l $1c(a0),a0` / `move.w $10(a0),d0` = `np->type` (NULL) then
`np->type->token`.

## Variant table (measured - see note under the table)

| variant | result |
|---|---|
| `func_2((l_9 != (3 \|\| g_10)));` (as above) | CRASH - inside mustty (MD.C) |
| `func_2((3 \|\| g_10));` (drop the `!=`) | CRASH - inside asn_check (call-arg type check) |
| `func_2((l_9 != (3 \|\| 4)));` (both operands constant) | compiles clean |
| `func_2((l_9 != (g_10 \|\| 3)));` (variable on the LEFT) | compiles clean |
| `char x = (3 \|\| g_10);` initializer, no call | compiles clean (observed during reduction; not persisted) |
| `l_9 = (3 \|\| g_10);` plain assignment, no call | compiles clean (observed during reduction; not persisted) |

Rows 1-4 are backed by persisted probes; rows 5-6 were observed clean during the
reduction tree walk - re-verify before relying on them.

So the trigger shape is: `||` with a CONSTANT on the left and a (const) GLOBAL
on the right, nested inside an expression passed as an ARGUMENT to a
PROTOTYPED function. The same guard covers `&&`; only `||` was exercised in
the reduced cases.

## Source-level analysis (v6.0 source distribution)

1. `binary_types()` (E2.C) contains, before the normal processing path:

```c
if (    (op eq AND or op eq OR)
      and (   np->left ->token eq ICON
           or np->right->token eq ICON
          )
     )
{
    ycanon(np);
    return;
}
```

2. The ONLY place that sets the `||`/`&&` node's type is AFTER this early
`return`: `np->type = CC_type(np->left, np->right);` (E2.C, in the
comparison/logical section near the `must2ty(np, R_SCALAR)` cases).

3. `ycanon()` collapses/rewrites the node; the collapsing branch
(`collaps()`) rewrites the node to `ICON` and frees the children but never
touches `np->type` - which was still NULL for the original `||` node.
Which of the ycanon branches leaves the type stranded for the exact
`(ICON || SYM)` shape was not fully traced - flagged as the open part of
this analysis - but the structural gap (early return past the sole type
assignment) is clear from the source.

4. Any later consumer that reads `np->type->token` without a NULL guard then
faults: `mustty()` (MD.C, `short tok = np->type->token;`), reached from
`must2ty()` for the `!=`/`==` operands, or `asn_check()` (MD.C, the argument
compatibility check `asn_check(tp, lp, FORPUSH)`) for the direct-argument
variant.

Suggested fixes (any one suffices): (a) in the guard, set a valid type after
`ycanon(np)` (e.g. the CC/int type the normal path would produce); (b) make
`collaps()` set `np->type` to a valid int/CC type; (c) defensively, have
`mustty()`/`asn_check()` report a compiler ICE diagnostic instead of
dereferencing NULL. (a)/(b) fix the root; (c) alone would still miscompile.

## Workaround

Avoid `(constant || variable)` / `(constant && variable)` as (part of) a call
argument - swap the operands (`variable || constant`), or hoist the `||` into
a separate named variable/assignment. The swap is measured to compile
correctly (persisted clean probe); the hoist variant is expected clean but
re-verify before relying on it.

## How to reproduce with an exception-aware debugger (emulator)

Under Hatari 2.x, enable exception debugging for bus errors, e.g. config
section `[Debugger]` key `nExceptionDebugMask` with the bus-error +
address-error bits (value 0x6; our value 0x40000006 additionally arms at
autostart), boot the
compiler normally; on the crash the CPU stops at the NEXT prefetched PC (the
trace's buspc names the faulting instruction) with the register dump, and
`--trace cpu_exception` logs
`fault ... addr 10 op 3028`. Without the debugger AHCC's own panic handler
prints `Fatal error ...` / writes a crash line. (A driver script that does
this fully headless is available on request.)

## Submission notes (for whoever forwards this)

- Upstream author Henk Robbers' website (members.chello.nl/h.robbers) has been
  offline for years; v6.0 (2020-11-27) was announced as possibly the final
  release, so a direct report may go unanswered.
- The distribution files (incl. the full source) are community-mirrored at
  https://github.com/swetland/ahcc (maintainer: Brian Swetland, Palo Alto, CA;
  activity visible on chaos.social/@swetland). A GitHub issue there is the
  most likely reachable venue for the source-level analysis above.

---

# Nederlandse versie

## Samenvatting

AHCC 6.0 (68000) crasht met een interne busfout tijdens het type-checken van
een logische OF-expressie van de vorm `(constante || global)` wanneer die
expressie als argument bij een aangeroepen functie met prototype staat. De
compiler sterft zonder enige C-foutmelding. 100% reproduceerbaar, onafhankelijk
van emulator of machine (geobserveerd onder Hatari met exception-debugging aan;
AHCC's eigen crash-handler print `Crash at text+00015f42`).

De oorzakelijke analyse hierboven wijst op de constant-vou-guard in
`binary_types()` (E2.C): voor `&&`/`||`-knopen met een `ICON`-operand roept die
`ycanon(np)` aan en `return`t vervolgens - waardoor de ENIGE regel die het
`->type` van de knoop zet (`np->type = CC_type(...)` later in dezelfde
functie) wordt overgeslagen. Wanneer zo'n knoop zonder type later `mustty()`
(MD.C) bereikt, dereferenceert de eerste regel `short tok = np->type->token;`
een NULL `->type` - busfout op adres $10.

## Minimale reproducer

Zie `MIN48.C` boven: een enkel .C-bestand, zonder bibliotheken; compileer het
als single-bestandsproject en AHCC sterft stil.

## Varianttabel (alles gemeten)

Zie de tabel boven: `(3 || g_10)` in een call-argument crasht (via `mustty`);
`(3 || g_10)` direct als argument crasht via `asn_check`; `(3 || 4)`,
`(g_10 || 3)`, initializer-zonder-call en gewone toewijzing-zonder-call
compileren proper. De trigger-vorm is dus: `||` met een CONSTANTS links en
een (constante) GLOBAL rechts, genest in een expressie als argument bij een
functie met prototype. Dezelfde guard dekt ook `&&`; alleen `||` is in de
teruggebrachte gevallen getest.

## Bronanalyse (v6.0 broncode-distributie)

Zie sectie "Source-level analysis" boven. Kort gezegd: de vroegtijdige
`return` na `ycanon(np)` in `binary_types()` omzeilt de enige
type-toewijzing; `collaps()` herschrijft de knoop naar `ICON` zonder
`np->type` te zetten; latere checkers (`mustty`, `asn_check`) hebben geen
NULL-guard op `np->type`. Suggestie voor de fix: stel na `ycanon(np)` (of in
`collaps()`) een geldig type in; een NULL-guard in `mustty()` is alleen een
noodreparatie (verbergt het echte probleem).

## Workaround

Vermijd `(constante || variabele)` / `(constante && variabele)` als
(deel van) een call-argument: verwissel de operanten (`variabele ||
constante`) of til de `||` naar een aparte benoemde variabele/tewijzing.
De verwisseling is gemeten als correct; de ge-hoiste variant wordt correct
verwacht, maar nog niet persistence-bewezen - verifieer voor gebruik.

## Indienen

De website van auteur Henk Robbers (members.chello.nl/h.robbers) is al jaren
offline; v6.0 (27-11-2020) werd aangekondigd als mogelijk de laatste release,
dus een direct bericht blijft wellicht onbeantwoord. De distributiebestanden
(inclusief volledige broncode) worden community-matig gespiegeld op
https://github.com/swetland/ahcc (onderhouder: Brian Swetland, Palo Alto, CA;
actief op chaos.social/@swetland). Een GitHub-issue daar is de meest
waarschijnlijk berebare plek voor deze analyse.
