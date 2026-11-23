# Guru Note-Marking Layer

Load this file for `notes`, `mark`, `known`, `--notes` flags, plain-terminal fallback, or when compiling session notes.

**Contents:** 1. Legend | 2. Rendering & Fallback | 3. What to Mark | 4. Anti-Inflation | 5. State Integration | 6. Commands | 7. Cornell Template | 8. Session Notes Block | 9. Example

---

## 1. Legend (Pure Markdown Default)

| Marker | Meaning | Use For |
| --- | --- | --- |
| *italic* | New term | First occurrence of a technical or domain term, paired with its meaning |
| **bold** | Core concept or keyword | The primary concept or mechanism being taught |
| `>` blockquote | Key rule or invariant | A high-value fact or rule worth recalling verbatim |
| `📌` | Copy-ready note | One self-contained takeaway line needing no extra context |
| `🔑` | Principle or design rule | Architectural invariants and engineering principles |
| `⚠️` | Pitfall or misconception | Common bugs, traps, or misleading assumptions |
| `🧠` | Mental model | An analogy that clarifies the underlying structure |

Translate descriptive labels (such as "Session Notes" or "Cue") into the user's language while keeping the symbols consistent. Never use raw HTML tags such as `<u>`.

## 2. Rendering & Plain-Text Fallback

Default mode (`render: markdown`) uses standard Markdown and emoji markers. If the user is in a plain terminal that does not render Markdown formatting cleanly, switch to `render: plain` in state (or in the State Block):

| Markdown Mode | Plain-Text Fallback |
| --- | --- |
| `>` key rule blockquote | Line prefixed with `>> ` |
| *italic* | `_italic_` |
| **bold** | `**bold**` (or `CAPS` for short keywords if asterisks are noisy) |

Emoji markers (`📌`, `🔑`, `⚠️`, `🧠`) remain usable in both modes.

## 3. What Deserves a Mark

Mark only: definitions, engineering rules, pitfalls, trade-off decisions, critical syntax or commands, and mental models. Never mark routine narration, obvious statements, or anything already marked in the same block.

## 4. Anti-Inflation Limits

If everything is highlighted, nothing stands out. Enforce these limits:

- Maximum 5 combined `📌` / `🔑` / `⚠️` markers per explanation block.
- One primary mark per concept (do not put the exact same sentence in both a `>` blockquote and a `📌` note).
- Never re-mark items listed in `state.known`.
- Density flags: `--notes off` (no markers) | `--notes min` (`📌` and `⚠️` only) | `--notes full` (full legend, default). `--tldr` outputs `📌` lines only.

## 5. Integration with Curriculum State

- Link each mark to a `concept_id` when curriculum state is active.
- `⚠️` items are added to `miss[]` if the user answers wrongly on that pitfall.
- `🔑` and `⚠️` items serve as prompt material for `revisit` flashcards (front: cue question, back: the note).
- Compile key takeaways at the end of a session into **📝 Session Notes** (<=7 lines).
- Items flagged via `known` are stored in `state.known` and excluded from future highlighting.

## 6. Commands

- **`notes [concept|session]`**: Output structured notes using the Cornell template below (defaults to the current session).
- **`mark [text]`**: Normalize the user's custom note into one clean line, link it to the nearest concept, and add it to session notes.
- **`known [item]`**: Record an item as already mastered so Guru stops highlighting or defining it.
- **`notes export`**: Save notes to `.guru/notes/<concept-id>.md` (only with user consent; see `references/curriculum.md` §9). In chat-only mode, output a single copy-paste Markdown block.

## 7. Cornell Template

```markdown
## <concept>

| Cue (Trigger Question) | Notes |
| --- | --- |
| When is automatic retry safe? | > Only when the operation is idempotent. |
| What is the risk on payment calls? | ⚠️ Retrying a POST without an idempotency key can double-charge users. |

**1-Sentence Summary:** Safe retries require idempotency, backoff, and a max-attempt cap.
```

## 8. Session Notes Block

```markdown
📝 Session Notes (N)
- ⚠️ ...
- 🔑 ...
- 📌 ...
```

Maximum 7 lines. Order: `⚠️` pitfalls first, then `🔑` principles, then `📌` quick takeaways.

## 9. Example

```markdown
*Idempotent* means running the same operation multiple times produces the exact same result as running it once.

> Automatic retries are only safe when the operation is idempotent.

⚠️ Retrying a payment `POST` request without an idempotency key can charge the customer twice.
📌 Safe retry = idempotency key + exponential backoff + max retry limit.

📝 Session Notes (2): idempotent | retry-safety
```
