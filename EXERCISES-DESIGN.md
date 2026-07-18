# Design: Exercise generation + evaluation for Learning Mode

Status: design stub on fork branch, pending upstream direction (see upstream issue).
Grounded in obsidian-yolo `main` @ 1.6.0.3.

## Existing infrastructure (verified in source)

| Piece | Where | State |
|---|---|---|
| File format | `<chapter>/exercises.md`, entries `## Title <!--ex:uuid8 kp:uuid8-->` + body | Parsed by `markdownScanner.ts` (`type 'ex'`), scanner flag `hasExercises` |
| View | `components/learning-view/ExercisesView.tsx` | Browse + practice modes, chapter/status filters, answer textarea |
| Missing | generator, practice-state persistence, answer evaluation | tab empty unless hand-written; "MVP no evaluation"; `practiced` resets on reload |
| Sibling pattern | `core/learning/generation/cardGenerator.ts` + `CARD_GENERATOR_PROMPT` + `projectWriter.ts` | The template to mirror |

## Slice 1: exerciseGenerator

New `src/core/learning/generation/exerciseGenerator.ts`, mirroring `cardGenerator`:

- Input per chapter: project topic, chapter title + contract, `knowledge.md` content, level, `exercises.md` target path.
- `EXERCISE_GENERATOR_PROMPT` (in `prompts.ts`): output pure markdown, one exercise per `##` heading with `<!--ex:<uuid> kp:<kpUuid>-->`, kp UUIDs copied from knowledge.md, sandwiched with `buildLanguageDirective`/`buildLanguageReminder` exactly like cards.
- Exercise mix (explicitly NOT recall — cards own recall): predict-the-output, spot-the-bug, apply-in-new-context, explain-back (teach a beginner), edge-case hunt. 1-2 per knowledge point, capped per chapter.
- Format detail: body = question only. Reference answer goes in an HTML comment `<!--answer: ... -->` or after a `---` separator (decide with maintainer; `---` matches the card convention and keeps the answer available to the evaluator without new parsing).
- Trigger: same flow as cards (OutlineBuilder / per-chapter action). Reuse `LEARNING_CARD_TOOL_NAMES` fs constraints.

## Slice 2: practice-state persistence

Smallest change consistent with codebase: a JSON store alongside the SRS store (`core/learning/srs/srsStore.ts` is the pattern): `{ exerciseUuid: { lastPracticedAt, attempts, lastVerdict } }`. ExercisesView reads it instead of hardcoding `practiced: false`.

## Slice 3: evaluation

On submit in practice mode:

- Prompt: exercise question + reference answer (if present) + bound KP body + user answer. Rubric: verdict `correct | partial | missed`, then 1-3 bullet gaps, in the project output language.
- Use `learningOptions.modelId` (same model as generation), stream result under the answer box.
- Optional (behind a setting): map verdict to SRS-style mastery contribution via `masteryMapping.ts`.

## Non-goals for v1

Grading rubrics per exercise type, multi-turn tutoring on a wrong answer, exercise difficulty adaptation, cross-chapter exams.

## Local fork usage note

If upstream declines, this can live in the fork as `feat/exercises-generation` and be rebased periodically; the file format is upstream-native so generated `exercises.md` stays compatible either way.
