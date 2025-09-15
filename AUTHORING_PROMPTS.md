# Buxo Authoring Prompts and Seeding Guide

This guide shows how to:
- Extract a course from PDFs/books if provided and turn it into a structured, high‑quality course in Buxo.
- Use a sequence of LLM prompts whose outputs feed the next step.
- Produce JSON that compiles into TypeScript seed scripts (split by chapters when needed).
- Run seeds to load the content into the database.

It is grounded in the current data model and API surface:
- Data model: `Course → Chapter → Section → Learning → Question → AnswerOption`.
- Reference: `backend/docs/openapi.yaml` (schemas and constraints) and `backend/src/scripts/seed.ts` (seeding pattern using TypeORM).


## 1) Data Model Summary (Authoring Scope)

- Course: metadata and catalog info
  - Required: `creatorId`, `categoryId`, `title`, `slug`, `bannerUrl`, `iconUrl`, `visibility`, `status`, `version` (numbers/strings per schema)
  - Useful: `summary`, `estMinutes`, `language` (default `en`), `tags`
- Chapter: ordered within a course
  - Fields: `courseId`, `ix`, `title`, `summary`
- Section: ordered within a chapter
  - Fields: `chapterId`, `ix`, `title`, `summary`, `learningCountCached`
- Learning: ordered within a section
  - Fields: `sectionId`, `ix`, `title`, `bodyMd`, `minQuestions` (>=2), `maxQuestions` (<=10), `quickReplies[]`, `resources?`, `state` (`draft|published`)
- Question: ordered within a learning
  - Fields: `learningId`, `ix`, `type` (`mcq|multi|short_text|long_text|true_false|ordering|match`), `promptMd`, `metadata?`, `difficulty? (easy|medium|hard)`, `rationaleMd?`
- AnswerOption: ordered within a question
  - Fields: `questionId`, `ix`, `contentMd`, `isCorrect`, `feedbackMd?`

Constraints to respect (from API spec):
- Indices `ix` are contiguous starting at 1 within their parent (chapter/section/learning/question).
- Each `Learning` should have 2–10 questions.
- For MCQ and TRUE/FALSE, ensure exactly one correct answer unless type requires multiple.
- `bannerUrl` and `iconUrl` must be valid URLs (you can upload via `/authoring/images` when available or use placeholders until final).


## 2) Authoring Workflow (End‑to‑End)

- Step A: Preprocess sources
  - Gather PDFs/chapters/book sections; extract table of contents (TOC), headings, abstracts.
  - Chunk by logical units (chapter → section) and keep page references.
  - Clean text: dehyphenation, remove headers/footers, preserve lists/code/math.
- Step B: Generate outline
  - Produce course → chapters → sections skeleton with ix ordering and concise summaries.
- Step C: Course metadata
  - Title, slug, summary, language, tags, estMinutes; banner/icon prompts or temporary URLs.
- Step D: Section learnings
  - For each section, produce one or more `Learning` items with well‑structured `bodyMd` and domain‑appropriate `quickReplies`.
- Step E: Questions and answers
  - For each learning, generate 2–10 questions across types with answer options, difficulty, feedback, rationale.
- Step F: Validate and fix
  - Run automated checks (indices, counts, schema) and issue targeted fix prompts where needed.
- Step G: Convert to seeds
  - Generate JSON (course plan by chapter). Optionally split by chapter for large courses.
  - Use a TypeScript seed template to ingest JSON and write to DB.
- Step H: Publish / iterate
  - Optionally use the API validate endpoint (if implemented); then publish.


## 3) Prompting Pipeline (Copy/Paste Ready)

Guiding principles for all prompts:
- Only use facts from the provided sources whenever provided; if missing, say “unknown” or propose options.
- Keep outputs deterministic: no extra prose; output in the exact JSON format.
- Keep `ix` contiguous starting at 1 at each level.
- Avoid token bloat: summarize where appropriate, but keep instructional clarity.

Set a persistent system message at the start of your LLM session:

System message (pin this at the top and reuse across steps):
“You are an expert course author for Buxo. Only use the provided sources. Produce strictly valid JSON per requested schema with no extra commentary. Prefer concise titles, actionable summaries, structured markdown in learnings, and pedagogically sound questions with rich, constructive feedback. Ensure indices `ix` start at 1 and are contiguous at every level. Assume English unless specified.”


### Prompt 1 — Course Outline (Chapters & Sections)

Input:
- Course sources: TOC excerpts, chapter titles, sections, or selected pages.
- Target learner profile (optional) and desired difficulty.

Instruction:
“Using the provided excerpts, propose a course outline with chapters and sections. Keep a balanced scope and avoid redundancy. Summaries should be 1–2 sentence, learner‑facing. Ensure `ix` starts at 1 and is contiguous at each level.”

Output (JSON, save as `course_outline.json`):
{
  "course": {
    "proposedTitle": "string",
    "proposedSlug": "kebab-case-lower",
    "summary": "string",
    "language": "en",
    "tags": ["string"],
    "estMinutes": 0
  },
  "chapters": [
    {
      "ix": 1,
      "title": "string",
      "summary": "string",
      "sections": [
        { "ix": 1, "title": "string", "summary": "string" }
      ]
    }
  ]
}

Notes:
- If the book already has chapters/sections, map them; otherwise derive a sensible outline.
- Keep names concise; avoid jargon unless defined.


### Prompt 2 — Course Metadata Details

Input:
- `course_outline.json`
- Catalog context: target `categoryId` (or a category slug to resolve later), branding direction for images.

Instruction:
“Refine and finalize course metadata. Provide title, slug, summary, tags, estMinutes, and concise banner/icon art directions (or placeholder URLs). Keep the slug globally unique (you may add a short random suffix placeholder).”

Output (JSON, save as `course_meta.json`):
{
  "title": "string",
  "slug": "kebab-case-lower",
  "summary": "string",
  "language": "en",
  "tags": ["string"],
  "estMinutes": 60,
  "banner": { "url": "https://..." , "alt": "string", "prompt": "(optional prompt for generation)" },
  "icon": { "url": "https://..." , "alt": "string", "prompt": "(optional prompt for generation)" }
}


### Prompt 3 — Section → Learning Drafts

Input:
- `course_outline.json`
- For each chapter/section, provide the relevant source text pages.

Instruction:
“For each section, generate one or more learnings (keep `ix` contiguous starting at 1). Each learning must include a clear title and a well‑structured `bodyMd` with: Overview, Key Concepts (bullets), Example(s), Common Pitfalls, and a short Summary. Provide `quickReplies` tuned to the content.”

Output (JSON per section, file name `learnings_ch{chapterIx}_sec{sectionIx}.json`):
{
  "section": { "chapterIx": 1, "sectionIx": 1, "title": "string" },
  "learnings": [
    {
      "ix": 1,
      "title": "string",
      "bodyMd": "# Title\n\n## Overview\n...",
      "minQuestions": 2,
      "maxQuestions": 6,
      "quickReplies": [
        "Yes, I understand",
        "Explain differently",
        "Give me an example",
        "Take Assessment"
      ],
      "resources": { "refs": [ { "source": "Book Name", "pages": "12-14" } ] },
      "state": "published"
    }
  ]
}

Notes:
- You may keep `minQuestions=2` and `maxQuestions` between 4–8 unless pedagogy demands more.
- `resources` can capture citations or links for later enrichment.


### Prompt 4 — Questions & Answers per Learning

Input:
- `learnings_ch{chapterIx}_sec{sectionIx}.json`
- Relevant source text for that learning.

Instruction:
“For each learning, generate 4–10 questions of varied types (`mcq`, `true_false`). Provide strong feedback for each option and an overall rationale. Ensure at least one MCQ per learning where applicable. Difficulty should skew from easy to medium, with a few hard items if warranted.”

Output (JSON per learning, file name `questions_ch{chapterIx}_sec{sectionIx}_l{learningIx}.json`):
{
  "learning": { "chapterIx": 1, "sectionIx": 1, "ix": 1, "title": "string" },
  "questions": [
    {
      "ix": 1,
      "type": "mcq",
      "promptMd": "Which statement is true about ...?",
      "difficulty": "easy",
      "rationaleMd": "This checks recall of ...",
      "metadata": { "shuffle": true },
      "answers": [
        { "ix": 1, "contentMd": "Correct answer", "isCorrect": true, "feedbackMd": "Nice work" },
        { "ix": 2, "contentMd": "Distractor A", "isCorrect": false, "feedbackMd": "Revisit the Key Concepts." }
      ]
    },
    {
      "ix": 2,
      "type": "true_false",
      "promptMd": "True or False: ...",
      "difficulty": "easy",
      "answers": [
        { "ix": 1, "contentMd": "True", "isCorrect": true, "feedbackMd": "Correct." },
        { "ix": 2, "contentMd": "False", "isCorrect": false, "feedbackMd": "Reason: ..." }
      ]
    }
  ]
}

Notes:
- For `multi`, mark multiple `isCorrect: true` and ensure feedback guides remediation.
- For `short_text/long_text`, include `metadata.expectedKeywords` or exemplar answer in `rationaleMd`.


### Prompt 5 — Validation & Fixes

Input:
- All JSON produced so far for a chapter.

Instruction:
“Validate against the constraints. Report issues and propose a corrected JSON patch (minimize changes). Typical checks: contiguous `ix`; each learning has 2–10 questions; MCQs have exactly one correct; feedback present; markdown headings present.”

Output (JSON):
{
  "valid": true,
  "issues": [],
  "patches": [
    { "file": "questions_ch1_sec1_l1.json", "description": "add feedback", "patch": "(unified or JSON merge)" }
  ]
}

You can apply small fixes interactively or regenerate the affected unit.


## 4) Final Authoring Package (Per Course)

For large courses, split by chapter to keep prompts and files manageable.

Recommended structure (created under the repo root `@coursegen`):
- `@coursegen/<slug>/course_outline.json`
- `@coursegen/<slug>/course_meta.json`
- `@coursegen/<slug>/ch<N>/learnings_ch<N>_sec<M>.json`
- `@coursegen/<slug>/ch<N>/questions_ch<N>_sec<M>_l<K>.json`

Optionally, produce a merged per‑chapter JSON for convenience when generating seeds:
- `@coursegen/<slug>/ch<N>/chapter<N>.json`:
{
  "chapter": { "ix": 1, "title": "...", "summary": "..." },
  "sections": [
    {
      "ix": 1,
      "title": "...",
      "summary": "...",
      "learnings": [
        {
          "ix": 1,
          "title": "...",
          "bodyMd": "...",
          "minQuestions": 2,
          "maxQuestions": 6,
          "quickReplies": ["Yes, I understand", "Explain differently", "Give me an example", "Take Assessment"],
          "state": "published",
          "questions": [ { "ix": 1, "type": "mcq", "promptMd": "...", "answers": [ ... ] } ]
        }
      ]
    }
  ]
}


## 5) Seed Generation (from JSON → DB)

We will convert the JSON to database rows through a TypeScript seed script that mirrors `backend/src/scripts/seed.ts`.

- Inputs available in repo:
  - Example seeding pattern: `backend/src/scripts/seed.ts`
  - Entities: `backend/src/authoring/entities/*.ts`
  - Script runner: `npm run seed` (runs `ts-node -r tsconfig-paths/register src/scripts/seed.ts`)

- Two common approaches:
  1) Duplicate `seed.ts` per course/chapter and paste the converted objects directly (quickest for one‑offs).
  2) Create a generic loader (recommended) that reads the merged JSON files from `@coursegen` and writes entities.

Below is a template for a generic loader script you can create (example name: `src/scripts/seed.from.json.ts`, inside the `backend/` folder). It assumes per‑chapter merged JSON files and a `course_meta.json` for top‑level metadata located in `@coursegen/<slug>`. You will pass the path to that folder as a CLI argument (e.g., `../@coursegen/<slug>` when running from `backend/`). Adapt paths as needed.

```ts
import 'dotenv/config';
import ds from '../data-source';
import { User } from '../users/user.entity';
import { Course } from '../authoring/entities/course.entity';
import { Chapter } from '../authoring/entities/chapter.entity';
import { Section } from '../authoring/entities/section.entity';
import { Learning } from '../authoring/entities/learning.entity';
import { Question } from '../authoring/entities/question.entity';
import { AnswerOption } from '../authoring/entities/answer-option.entity';
import { CATEGORIES } from '../catalog/categories.constants';
import { readFileSync, readdirSync } from 'fs';
import { join } from 'path';

async function main() {
  await ds.initialize();
  const users = ds.getRepository(User);
  const courses = ds.getRepository(Course);
  const chapters = ds.getRepository(Chapter);
  const sections = ds.getRepository(Section);
  const learnings = ds.getRepository(Learning);
  const questions = ds.getRepository(Question);
  const answers = ds.getRepository(AnswerOption);

  const seedEmail = process.env.SEED_USER_EMAIL || 'seed@example.com';
  let creator = await users.findOne({ where: { email: seedEmail } });
  if (!creator) {
    creator = await users.save(users.create({ email: seedEmail, name: 'Seed User', isProfileComplete: true, country: 'US' }));
  }

  // Resolve category (by index, id, or your own mapping)
  const categoryId = CATEGORIES[0].id; // TODO: replace based on your course

  // Load meta
  const root = process.argv[2]; // e.g., ../@coursegen/<slug> when running from backend/
  const meta = JSON.parse(readFileSync(join(root, 'course_meta.json'), 'utf8'));

  let course = await courses.save(courses.create({
    creatorId: creator.id,
    categoryId,
    title: meta.title,
    slug: meta.slug,
    summary: meta.summary,
    bannerUrl: meta.banner?.url || 'https://placehold.co/1200x630',
    iconUrl: meta.icon?.url || 'https://placehold.co/512x512',
    visibility: 'public',
    status: 'published',
    estMinutes: meta.estMinutes ?? 60,
    language: meta.language || 'en',
    tags: meta.tags || [],
    version: 1,
  }));

  // Load chapters (sorted by chapter index from folder names like ch1, ch2, ...)
  const chapterDirs = readdirSync(root).filter((d) => /^ch\d+$/.test(d)).sort((a,b) => parseInt(a.slice(2)) - parseInt(b.slice(2)));
  for (const chDir of chapterDirs) {
    const chapterJson = JSON.parse(readFileSync(join(root, chDir, `chapter${chDir.slice(2)}.json`), 'utf8'));

    let ch = await chapters.save(chapters.create({ courseId: course.id, ix: chapterJson.chapter.ix, title: chapterJson.chapter.title, summary: chapterJson.chapter.summary }));

    for (const s of chapterJson.sections) {
      let sec = await sections.save(sections.create({ chapterId: ch.id, ix: s.ix, title: s.title, summary: s.summary, learningCountCached: 0 }));

      for (const l of s.learnings) {
        let learn = await learnings.save(learnings.create({
          sectionId: sec.id,
          ix: l.ix,
          title: l.title,
          bodyMd: l.bodyMd,
          minQuestions: l.minQuestions ?? 2,
          maxQuestions: l.maxQuestions ?? 10,
          quickReplies: l.quickReplies ?? ["Yes, I understand","Can you explain more?","Show example","Take Assessment"],
          state: l.state ?? 'published',
          resources: l.resources ?? null,
        }));

        if (Array.isArray(l.questions)) {
          for (const q of l.questions) {
            let qq = await questions.save(questions.create({
              learningId: learn.id,
              ix: q.ix,
              type: q.type,
              promptMd: q.promptMd,
              metadata: q.metadata ?? null,
              difficulty: q.difficulty ?? null,
              rationaleMd: q.rationaleMd ?? null,
            }));

            if (Array.isArray(q.answers)) {
              for (const a of q.answers) {
                await answers.save(answers.create({ questionId: qq.id, ix: a.ix, contentMd: a.contentMd, isCorrect: a.isCorrect, feedbackMd: a.feedbackMd ?? null }));
              }
            }
          }
        }

        // update section cached count
        sec.learningCountCached = (sec.learningCountCached || 0) + 1;
        await sections.save(sec);
      }
    }
  }

  await ds.destroy();
}

main().catch(async (err) => { console.error(err); try { await ds.destroy(); } catch {} process.exit(1); });
```


## 6) Running the Seeds

- Prerequisites
  - Configure DB in `backend/.env` and ensure the app can connect.
  - Optionally set `SEED_USER_EMAIL` in `backend/.env` to attribute content to a specific creator.
  - Install deps and prepare backend:
    - `cd backend`
    - `npm i`
    - If needed, run migrations: `npm run migration:run`

- One‑off approach (copy from existing seed)
  - Copy `src/scripts/seed.ts` to a new name (e.g., `seed.<slug>.ts`), replace the inline objects with your generated content, and run:
    - `npm run seed -- -r tsconfig-paths/register src/scripts/seed.<slug>.ts` (or add a new npm script)

- Generic loader approach (recommended)
  - Create `src/scripts/seed.from.json.ts` in `backend/` based on the template above.
  - Place your authored JSON under `@coursegen/<slug>/...` (as described in Section 4).
  - Add an npm script (recommended): in `backend/package.json` add
    - `"seed:from-json": "ts-node -r tsconfig-paths/register src/scripts/seed.from.json.ts"`
  - Run either (from `backend/`):
    - `npm run seed:from-json -- ../@coursegen/<slug>`
    - or directly: `npx ts-node -r tsconfig-paths/register src/scripts/seed.from.json.ts ../@coursegen/<slug>`

- After seeding
  - Optional validation (if implemented): `POST /authoring/courses/{courseId}/validate`.
  - Optional publish step: `POST /authoring/courses/{courseId}/publish`.


## 7) Quality Checklist (Before Seeding)

- Indices: `ix` in chapters/sections/learnings/questions start at 1, contiguous, match file ordering.
- Question counts: each learning has 2–10 questions; MCQ has exactly one correct option.
- Feedback: each answer option has constructive `feedbackMd`.
- Markdown: `bodyMd` uses clear headings, bullets, examples, and short summary.
- Consistency: titles and summaries are concise, learner‑facing, avoid unexplained jargon.
- Assets: `bannerUrl` and `iconUrl` are valid URLs (or use uploader endpoint later).
- Tags & language: set appropriately; `estMinutes` makes sense for the overall scope.


## 8) Example Mini Course (All Steps Summarized)

Outline (`course_outline.json`):
```json
{
  "course": {
    "proposedTitle": "Data Visualization Basics",
    "proposedSlug": "data-visualization-basics",
    "summary": "Learn the core principles and practices of data visualization.",
    "language": "en",
    "tags": ["data", "visualization", "charts"],
    "estMinutes": 90
  },
  "chapters": [
    {
      "ix": 1,
      "title": "Foundations",
      "summary": "Core concepts and good practices.",
      "sections": [
        { "ix": 1, "title": "Why Visualize?", "summary": "Purpose and outcomes." },
        { "ix": 2, "title": "Choosing Encodings", "summary": "Match data to marks." }
      ]
    }
  ]
}
```

Metadata (`course_meta.json`):
```json
{
  "title": "Data Visualization Basics",
  "slug": "data-visualization-basics-xyz1",
  "summary": "Learn the core principles and practices of data visualization.",
  "language": "en",
  "tags": ["data", "visualization", "charts"],
  "estMinutes": 90,
  "banner": { "url": "https://placehold.co/1200x630", "alt": "charts collage" },
  "icon": { "url": "https://placehold.co/512x512", "alt": "bar chart icon" }
}
```

Merged chapter (`@coursegen/data-visualization-basics/ch1/chapter1.json`):
```json
{
  "chapter": { "ix": 1, "title": "Foundations", "summary": "Core concepts and good practices." },
  "sections": [
    {
      "ix": 1,
      "title": "Why Visualize?",
      "summary": "Purpose and outcomes.",
      "learnings": [
        {
          "ix": 1,
          "title": "Goals of Visualization",
          "bodyMd": "# Goals of Visualization\n\n## Overview\nWe visualize to communicate and to explore...",
          "minQuestions": 2,
          "maxQuestions": 6,
          "quickReplies": ["Yes, I understand", "Explain differently", "Give me an example", "Take Assessment"],
          "state": "published",
          "questions": [
            {
              "ix": 1,
              "type": "mcq",
              "promptMd": "Which is a primary goal of data visualization?",
              "difficulty": "easy",
              "answers": [
                { "ix": 1, "contentMd": "To communicate insights", "isCorrect": true, "feedbackMd": "Exactly—communication is central." },
                { "ix": 2, "contentMd": "To increase file size", "isCorrect": false, "feedbackMd": "Not a goal—this is irrelevant." }
              ]
            },
            {
              "ix": 2,
              "type": "true_false",
              "promptMd": "True or False: Visualization can aid exploration.",
              "difficulty": "easy",
              "answers": [
                { "ix": 1, "contentMd": "True", "isCorrect": true, "feedbackMd": "Correct—exploration is a key use." },
                { "ix": 2, "contentMd": "False", "isCorrect": false, "feedbackMd": "Revisit the Overview section." }
              ]
            }
          ]
        }
      ]
    }
  ]
}
```

Seeding: create `src/scripts/seed.from.json.ts` in `backend/` from the template, place the files under `@coursegen/data-visualization-basics`, then run:
- `cd backend`
- `npm run seed:from-json -- ../@coursegen/data-visualization-basics`


## 9) Tips & Guardrails for Better Outputs

- Prefer concrete examples and analogies; add a short scenario where possible.
- Keep questions aligned to the learning objectives; avoid trivia.
- Balance difficulty; include remediation hints in `feedbackMd`.
- Maintain consistent voice and heading structure in `bodyMd` for readability.
- When unsure, add a TODO in `resources` and flag for SME review rather than guessing.


## 10) Where This Maps in the Code

- Authoring DTOs and constraints are documented in `backend/docs/openapi.yaml` (e.g., `LearningUpdateDto`, question types and difficulty, min/max question counts).
- The seeding pattern and entity relationships are demonstrated in `backend/src/scripts/seed.ts` and `backend/src/authoring/entities/*.ts`.
- Use these as the source of truth while adjusting the prompt outputs.

---
With this pipeline, you can reliably convert books/PDFs into Buxo courses via LLM‑assisted authoring, validate the structure, and seed the database for delivery.
