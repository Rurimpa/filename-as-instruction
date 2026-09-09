# Filename as Instruction

**Put the reading constraint in the filename, not in the file.**

A one-step technique for making an AI coding agent stop skim-reading a document
it must read in full. Costs nothing, works before the file is opened, and
survives across tools.

日本語版：[README.ja.md](README.ja.md)

---

## The problem

Coding agents read files partially and then act with full confidence.

This is well documented. Read tools truncate. Attention favours the edges of a
long context. Agent instruction files get silently cut (Codex concatenates
`AGENTS.md` files and truncates the combined payload; skill listings truncate
descriptions). After compaction, agents drift back to sampling files with
`grep` instead of reading them.

The failure is invisible from the inside. The file is on disk. The editor shows
it. The agent read a subset and had no way to tell you which part.

## The usual fixes, and where they sit

Every widely-shared fix lands in one of two places:

- **Inside the file** — "read this to the end", `Do not summarize`, a preamble
  at the top.
- **Somewhere else** — a rule in `CLAUDE.md` / `AGENTS.md`, a `PreToolUse` hook,
  a `Stop` hook that catches "I read the whole file" claims, a skill.

Both categories share a gap. The in-file instruction **only fires once the file
is open** — and a partial read may never reach the line that says "read it all".
The external rule fires, but it lives in a different place and has to be
maintained there.

## The technique

Put the constraint in the **filename**.

```
design-philosophy.md
→ design-philosophy(do-not-read-in-part).md
```

That's the whole thing. One `mv`.

The filename is in the agent's context from the moment it lists the directory,
resolves a path, or receives a tool result — **before** any decision about how
much of the file to read. There is no partial read that skips it.

## Why the position matters

| | fires when | can a partial read miss it? |
|---|---|---|
| Preamble at top of file | after opening | **yes** — open lines 400-450 and it never loads |
| `CLAUDE.md` rule | session start | no, but costs context every turn |
| `PreToolUse` hook | before first tool | no — needs config + script |
| **Filename** | **before opening** | **no** |

## What we observed

Three cases from a long-running agent workspace. Each is **n=1**. This is field
observation, not a controlled experiment.

**Case 1 — the preamble could not fire.** A session was asked to read one short
section of a document. It opened an eight-line range. The document's opening
lines contain a strong instruction to read to the end. Those lines were never in
its context. The instruction was well written and structurally unable to apply.

**Case 2 — the name fired before opening.** The same document was renamed to
carry the constraint. A later session, asked for a partial read, refused
*before* issuing any read call, citing the filename as its reason.

**Case 3 — what "skim" actually means.** A session instructed to skim reported
that skimming is not an available action for it. It can reduce *how much* it
reads, or reduce *how carefully* it reads. Block the first with the filename and
the second with an in-file preamble, and full attentive reading is the only
remaining option.

## Cost

| approach | steps to set up | portable across tools |
|---|---|---|
| **Filename** | **1** (`mv`) | **yes, unchanged** |
| Preamble in file | 1 per file | yes |
| `CLAUDE.md` rule | 1 + context cost every turn | rewrite per tool |
| `PreToolUse` hook | config + script + debugging | rewrite per tool |
| Verification hook | script + test suite | rewrite per tool |

## Limits

Read these before adopting it.

- **No enforcement.** A filename persuades; a hook blocks. If you need a
  guarantee, you need a hook. This is the cheapest outer layer, not a
  replacement for one.
- **It does not reach quoters.** If another agent hands you an excerpt, you never
  see the filename. The constraint travels with the file, not with its contents.
- **Renaming breaks references.** Every path that names the file has to be
  updated, and old references in logs and archives should *not* be rewritten.
- **It depends on scarcity.** Name every file this way and the signal stops
  standing out.
- **Environment-dependent.** In some upload-based interfaces the model is not
  reliably shown the filename at all. Verify in your own setup before relying
  on it.
- **Same mechanism as an attack.** Filename-borne prompt injection is a known
  threat class (OWASP LLM01). This is that mechanism, pointed at your own files.
  If you accept filenames from untrusted sources, keep sanitising them.

## Prior art

The parts are all known. We are not claiming the parts.

- **Filenames as agent-facing metadata.** Purpose prefixes (`AI-handoff_...`),
  ISO-8601 date prefixes, numeric ordering prefixes, semantic slugs for
  retrieval. Widely practised. What these carry is *what the file is* — not
  *how it must be read*.
- **Constraining how a document is read.** "Do not summarize. Do not
  paraphrase." "Do not batch-load entire folders." Also practised — and
  consistently placed in the file body or an adjacent control document, which
  means it fires only after opening.
- **Filenames as an instruction position.** Established on the *attack* side:
  filename-borne prompt injection, suspicious-filename detectors, OWASP LLM01
  mitigations that recommend never putting a raw filename into an instruction
  position.
- **What we did not find**, across roughly 40 English and Japanese sources: the
  first two combined — a filename carrying a *reading constraint*, used
  defensively on one's own files. Absence in that sample is not proof of
  absence. If you know of prior work, please open an issue; we will credit it
  here.

## License

MIT. Use it, rename your files, no attribution required.
