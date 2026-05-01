# Fix `textDocument/completion` race against `textDocument/didChange`

This PR fixes a long-standing race in `LtexTextDocumentService` that
caused `textDocument/completion` to return suggestions for the wrong
prefix when the client triggers completion immediately on every
keystroke.

## What goes wrong

When a user types continuously and the client requests completion on
each character, the popup shows results that have nothing to do with
the word being typed. The candidate list either stays stuck on a
prior word, or returns the alphabetical first slice of every word
starting with a single early letter — typing `Struc` returns words
beginning with `S`; typing `info` returns the same `s*` list left
over from earlier in the document.

The bug is invisible in clients that manually trigger completion (VS
Code with manual `Ctrl+Space`) or in clients that debounce or serialize
completion and then spellchecking (I presume this is how VS Code
internally works).  The bug becomes deterministic and triggers every
completion in setups that fire completion immediately on each keystroke
(Emacs corfu with `corfu-auto-delay 0.01` / `corfu-auto-prefix 1`).

## Why it happens

### The mechanics first

When the user types a character in an LSP-aware editor, the editor
sends two LSP messages to the language server in quick succession:

1. **`textDocument/didChange` notification.** The server keeps its
   own copy of every open document so it can run grammar checks and
   answer queries. This notification tells the server "the document
   just changed; here is the new text" so the server can update its
   copy.
2. **`textDocument/completion` request.** This asks the server "the
   cursor is at line *L*, character *C*; what words might the user
   be typing?" The server looks up the position in its copy of the
   document, walks back from there to find the partial word the user
   has typed (the *prefix*), and returns matching dictionary entries.

For the completion to be useful, the server must answer the second
message based on the document state established by the first. The
editor relies on this: it sends them in the right order on the wire,
`didChange` then `completion`.

### What goes wrong

LTeX+ does not actually update its copy of the document when
`didChange` is processed. Instead, it queues the update on a
background single-thread worker (used for slow work like grammar
checks) and returns immediately:

```kotlin
override fun didChange(params: DidChangeTextDocumentParams) {
    ...
    this.languageServer.singleThreadExecutorService.execute {
      document.applyTextChangeEvents(params.contentChanges)
      ...
    }
}
```

The block inside `.execute { ... }` is *queued for later*. The
handler itself returns to the LSP framework as soon as the work is
queued — *before* the server's copy of the document has actually
been updated.

`textDocument/completion`, by contrast, was handled inline: the
handler immediately read `document.text`, computed the prefix at the
requested position, and returned the result. Because `lsp4j` (the
server-side LSP framework that ltex-ls-plus is built on) processes
incoming messages one after another, it dispatches `completion` as
soon as `didChange`'s handler has returned — but the queued update
has not run yet. So `completion` reads the *previous* version of
the document.

The order of operations on the server therefore looks like:

1. `didChange` arrives → text update is queued for the worker → handler returns.
2. `completion` arrives → handler runs immediately → reads `document.text`, still the previous version.
3. Eventually the worker dequeues the update and applies it. The next `completion` is again one keystroke behind.

The popup never catches up to what the user is typing.

Why this has more severe consequences than "one request behind"
suggests: each response is returned with `isIncomplete: false`, which in
LSP semantics means "the list is exhaustive for this prefix; you may
refilter it locally without asking again." This is correct behavior in
itself — for ltex-ls-plus's strict-prefix matching, the list *is*
exhaustive for the prefix the server computed, so `isIncomplete: false`
is not a bug. But combined with the race it amplifies the damage the
user sees: clients that respect the hint stop sending new requests for
the rest of the typing session, so the stale list lingers for the entire
word the user is typing. And in `checkFrequency: edit` mode, the worker
can queue multiple didChange notifications behind a slow grammar pass,
so the server's view of the document can fall many keystrokes behind —
not just one.

### A secondary inconsistency

To translate an LSP `Position(line, character)` into the byte offset
needed to find the prefix, the document keeps both the text itself
and a list of line-start byte offsets. When the text changes, both
have to be updated. But they're updated by two separate calls
(`super.setText(...)` followed by `reinitializeLineStartPosList(...)`)
with no protection between them. A reader looking at the document
mid-update can see new text paired with old offsets, computing a
byte offset that points into completely unrelated text.

## The fix

Two changes, both small.

**1. Apply text changes immediately, not later.** `didChange` now
performs `applyTextChangeEvents` on the same thread that received the
notification, so by the time the handler returns the document state is
already up-to-date. Only the slow grammar / diagnostics pass remains
on the background worker. This is the standard pattern used by other
lsp4j servers (texlab, rust-analyzer): text mutation is fast enough to
handle inline; only the domain-heavy follow-up work is queued.

```kotlin
override fun didChange(params: DidChangeTextDocumentParams) {
    val document: LtexTextDocumentItem = getDocument(uri) ?: return
    if (document.beingChecked) document.cancelCheck()

    // Apply synchronously so subsequent requests see the new state.
    document.applyTextChangeEvents(params.contentChanges)
    document.version = params.textDocument.version

    // Slow grammar pass stays on the worker.
    if (settings.checkFrequency == Settings.CheckFrequency.Edit) {
      this.languageServer.singleThreadExecutorService.execute { ... }
    }
}
```

**2. Mark the document state mutators and accessors `@Synchronized`.**
This guarantees that anyone reading the document text or computing a
position cannot see a half-updated state where the text and the
line-offset list are out of sync. The completion path captures both
under one `@Synchronized` block, then releases the lock before doing
the slow work (fragmentization, dictionary scan) on the captured
snapshot:

```kotlin
val (code: String, pos: Int) = synchronized(document) {
  Pair(document.text, document.convertPosition(position))
}
// ...fragmentize, prefix-walk, dictionary scan run on the snapshot...
```

`completion` itself is reverted to its original synchronous shape — no
longer routed through `singleThreadExecutorService`, since the
synchronization above is now sufficient and routing through the
executor would unnecessarily queue completion requests behind grammar
passes (visible latency hit in `checkFrequency: edit` mode).

## What this affects

- **Threading model.** Text mutation is now synchronous on the dispatch
  thread; only diagnostics work runs on the worker. This is what most
  lsp4j servers already do.
- **Behavior under all clients.** Existing behavior is preserved for
  clients that already worked. Clients that fire completion on every
  keystroke (the original bug surface) now see correct results.
- **No protocol change**, no new settings, no behavioral switch.

## Testing

Existing test suite passes (236 tests, 2 pre-existing skipped).

A new regression test was added to `LtexTextDocumentServiceTest`:
`testDidChangeAppliesSynchronouslyEvenWhenExecutorBusy`. It blocks the
worker with a `CountDownLatch`, fires `didChange`, and asserts the
document text reflects the change. With the previous implementation
the assertion fails (the queued update is stuck behind the blocked
worker); with this fix it passes.

Manual reproduction in Emacs (corfu, `corfu-auto-delay 0.0`,
`corfu-auto-prefix 1`):

| Prefix typed | Before fix              | After fix              |
|--------------|-------------------------|------------------------|
| `Dir`        | stale `s*` results      | `Dirac`, `DirectX`, … |
| `Struc`      | unchanged from `S` list | only `Struc*` words    |
| `info`       | `s*` words from earlier | `informant`, …         |

VS Code users will see no behavior change. The bug was already
invisible there in every configuration we tested, so this PR is
correctness for clients that *did* expose it (Emacs corfu, and
likely other clients that fire completion immediately on each
keystroke).
