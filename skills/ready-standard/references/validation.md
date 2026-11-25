# Validation

## Run it

The validator for a Ready Logic Package (RLP) is Ready Room's loader,
published as the `@readyroom/ready` package (Node 24 or later). `<repo>` is the
directory that contains `product/`.

```sh
npx @readyroom/ready validate <repo>
```

Inside a Ready Room checkout, `npm run ready -- validate <repo>` runs the same
code from source; Ready Room validates its own package that way. The CLI runs
the `strict` validation profile (all schema, graph, completeness, and editorial
checks). Ready Room's programmatic loader (`loadRecords` / `loadModel`) also
supports an `integrity` profile that runs structural and graph-integrity errors
only during incremental authoring.

Output is one line per issue, then a summary:

```
product/shared/data/notes.ready: warning entity.index Column owner_id badge I9·1 names no index
24 records: 1 product, 1 capability, 2 flows, …; 1 supplementary file
0 errors, 1 warning
```

The command exits 1 when there is any error. A change is not done until it
reports **0 errors**. Treat each warning as a real problem to fix or to raise
with the owner; do not add new warnings silently.

The codes and their meanings are listed in [format.md](format.md#validation).

## Fix in this order

A record that fails its shape check is not loaded at all, so every record that
references it then reports `ref.missing`. Work from the top:

1. **File problems:** `yaml.parse`, `record.shape`, `record.version`,
   `record.kind`, `record.id`. Usually unquoted `: ` in text, a tab, a
   duplicate key, or `ready` not `3`.
2. **Field problems:** `field.invalid`, `field.unknown`, `interface.fields`.
   Check the field against [format.md](format.md) or
   `npx @readyroom/ready schema`. Common causes: a number where text is
   required (`value: 2000`, an unquoted `format_version`), an empty string, a
   misspelled or invented field, a value outside an enum, a display id where
   an id is required, a surface field on an operation, or a file path not
   under `product/`.
3. **Identity problems:** `record.duplicate`, `display.duplicate`,
   `product.count`, then `part.mismatch`.
4. **Re-run.** Many `ref.missing` errors disappear once the records above load.
5. **Graph problems:** `ref.missing` (a typo, a display id used where an id is
   required, a renamed id still cited, or a record that was removed),
   `ref.kind` (an edge's `to` pointing at a flow, a message's `interface` at a
   surface), `ref.file` (the file must exist in the package with an allowed
   extension).
6. **Homes:** `home.capability` (a flow or sequence with no `capability`, or
   one that does not exist), then `home.shared` (a
   capability-local rule used elsewhere: move it to shared).
7. **Flows:** `flow.subject`, `flow.empty`, `step.flow`, `step.kind`. Fix the
   flow's subject first, since it decides whether its steps are journey steps
   or `state` steps.
8. **Edges:** `edge.trigger` (add or correct the trigger record), then
   `edge.sequence` (the edge's `outcome` must be one of its sequence's
   `outcomes`), then `edge.start` (all edges that run one sequence leave one
   step with one trigger).
9. **Sequences and rules:** `sequence.participant`, `ops.target`,
   `rule.empty`, `rule.orphan`, `rule.decision`, `entity.column`.
10. **Reachability warnings:** `step.unreachable`, `sequence.outcome`,
    `sequence.unreached`, `interface.unused`, `trigger.unused`. Each means a
    part of the graph leads nowhere or nothing leads to it. Find the real
    entry, edge, or trigger; if there is none, ask the owner whether the
    record is still intended.
11. **Data warnings:** `ops.index` and `entity.index` (a badge with no index,
    or `I1.1` written with a period instead of `·`), `table.shape` (a rule
    table row with the wrong number of cells), `data.state` (a step's data
    `state` that is not the id of a `state` step in that entity's lifecycle).
12. **Plain-words warnings:** `text.code` (a prose field reads as code; the
    message names the match and where it belongs), `text.id` (a prose field
    names a record by id; link it from the typed field), `text.call` (a
    message's text repeats its `call`), `text.ref-id` (a graph node's text
    repeats a ref), `summary.missing`, `summary.long`, `summary.repeats` (say
    what the record is or does, not its title again), `name.long`,
    `outcome.long`. Fix these by moving the identifier into its typed field
    and writing the sentence in words; see
    [authoring.md](authoring.md#write-for-the-returning-reader). Work
    capability by capability, worst records first, and never satisfy a
    warning by deleting the detail the owner cares about.
13. **Empty screen steps:** `step.empty`. Write what the person sees or does
    at this moment in `sees` or `does`, or the words they read in `copy`, from
    the owner's intent, the design, or the product itself. Never invent copy or
    behavior to silence it: if what the person sees or does is not known, it is
    a question for the owner, and the warning stays until it is answered.
14. **One-home warnings:** `text.history`, `sees.branch`, `sees.platform`,
    `term.unused`, `notes.long`, `column.restates`, `copy.repeated`,
    `text.repeated`. Move each fact to its home (see
    [mistakes.md](mistakes.md#one-home-per-fact)):
    - the reason to `rationale` without its date, the condition to an edge's
      `guard`, the client to `platforms`, the word to `copy`;
    - `notes.long`: behavior to its typed fields, long reference prose to a
      supplementary file, keeping only context and pointers in `notes`;
    - `column.restates`: nullability, keys, foreign keys, defaults, and checks
      to their typed column fields, and the `rule` opens with what the column
      means;
    - `copy.repeated`: drop the surface's words from the step's `copy` and
      keep only what this step alone shows;
    - `text.repeated`: keep the sentence in the record that owns the fact and
      link that record from the others.
15. **States-and-moves warnings:** `step.trigger-title`, `step.action`,
    `decision.action`, `decision.branch`, `edge.both-ways`. Rename a step whose
    title repeats its own trigger; replace an `action` step with a `task`,
    `sequence`, or `screen`/`state`; turn an unowned UI-choice `decision` into
    a `screen` (or set `owner` if it is a multi-destination human review
    decision); name each non-action decision branch with a guard, label, or
    outcome; and give the way back its own trigger. Keep ids stable; change
    titles and trigger names (see [mistakes.md](mistakes.md#flows-and-edges)).
16. **DCL warnings:** `dcl.same` means a capability repeats the product
    default's level. Remove the capability's duplicate `dcl` so it uses the
    default; a separate basis alone does not justify repeating the level.
17. **Size and file warnings:** `size.flow` and `size.sequence`: split into
    lettered parts at a phase boundary (see
    [ids-and-parts.md](ids-and-parts.md#lettered-parts)). `record.large`:
    split the record, or move its long lists and examples to a supplementary
    file in `files`. `file.unreferenced`: list the file in `files` on the
    record it supports, or remove it.

Fix the cause, not the symptom: do not satisfy `rule.orphan` by listing the
rule on an unrelated record, `sequence.unreached` by giving an edge-run
sequence a trigger, or `sequence.outcome` by adding an edge to a step the
product never reaches. Find the process that actually applies, runs, or ends
it; if there is none, ask the owner whether the logic is still intended.

## What the validator does not check

Passing validation means the graph is well formed, not that the intent is
right. It does not check:

- that the content is present tense and free of history and task language;
- that a rule is applied by a process rather than merely referenced;
- that capabilities are outcomes rather than channels, teams, or layers;
- that a shared record is really shared, or a local one really local, beyond
  `home.shared`;
- that a trigger's name, an edge's guard, and a sequence's outcomes describe
  what really happens;
- that logic lives in typed fields rather than `notes`, beyond the length of
  `notes`;
- that copy is verbatim, or that a consumer does not restate its owner in
  other words: `copy.repeated` and `text.repeated` find only exact repeats;
- that a record gives every reason it should; a missing `rationale` is not
  reported, because a reason is never invented to satisfy a check;
- that the RL matches the code.

Those are the reviewer's job; see [principles.md](principles.md) and
[mistakes.md](mistakes.md).

## In CI

A repository with a ready package should run the validator in CI (Ready Room
runs it, and `app/test/intent-readyroom.test.ts` checks the same). Any other
repository adds a job step `npx @readyroom/ready@<version> validate .`, pinning
the version so a format change lands as a deliberate upgrade.
