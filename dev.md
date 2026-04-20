# Concurrent daemon mode — dev status

**Branch:** `feat/multi-session-daemon`
**HEAD:** `f2c90e9` — *Support concurrent clients in --endless TCP daemon mode (POC)*
**Base:** `fork/develop`
**Status:** Proof-of-principle merged into the branch; verification done at the "can two clients connect at once" level. Not yet production-ready.

---

## What shipped in `f2c90e9`

One-file change to `src/main/kotlin/org/bsplines/ltexls/LtexLanguageServerLauncher.kt` (70 insertions, 9 deletions). Under `--server-type=tcpSocket --endless`, the accept loop is now thread-per-connection instead of strictly serial:

```
call()
  └── if (endless && tcpSocket) → runConcurrentAcceptLoop(...)
      else                       → do { launchServer(...) } while (endless)   // unchanged

runConcurrentAcceptLoop(serverSocket, logOutputStream, port):
    while (true):
        val client = serverSocket.accept()
        Thread({ runTcpSession(client, logOutputStream) }, "ltex-session-$id").start()

runTcpSession(clientSocket, logOutputStream):
    // stream setup + optional tee + launch(in, out) + finally close socket
```

Per-session isolation is inherited for free: the companion-object `launch()` (`LtexLanguageServerLauncher.kt:255`) already constructs a fresh `LtexLanguageServer` per call, which in turn owns its own `SettingsManager`, `LtexTextDocumentService` document map, `LanguageToolInterface`, and `JLanguageTool` instance. No changes were needed in any of those files.

Stdio mode and single-shot TCP mode (`--server-type=tcpSocket` without `--endless`) are unchanged.

## Verification done so far

| Check | Outcome |
|---|---|
| `mvn package -DskipTests` | Clean build |
| `mvn test -Dtest=LtexLanguageServerLauncherTcpSocketTest` | Passes (existing single-shot TCP test — no regression) |
| Daemon starts, logs `Waiting for client to connect on port N` from `runConcurrentAcceptLoop` | ✓ |
| Two concurrent Python TCP connects succeed immediately (no kernel-backlog queuing of the second) | ✓ |
| `jstack <daemon-pid>` shows `ltex-session-1` and `ltex-session-2` as independent daemon threads during concurrent load | ✓ |
| A third connection after the first two close is accepted (loop continues) | ✓ |
| Daemon exits cleanly on SIGINT | ✓ |

The manual smoke test used raw TCP connections (Python `socket.create_connection`). No actual LSP `initialize`/`didOpen`/diagnostic flow was exercised — we only validated that the *accept-and-thread-spawn* plumbing works. See hazard #7 below.

---

## Deferred hazards — what an agent picking this up needs to verify next

Priority roughly descending. Each item includes a reproduction approach, a "done" criterion, and a likely mitigation sketch. Do not assume any of these are safe without running the reproduction first — the POC audit identified them as theoretical risks, not empirically confirmed bugs.

### 1. `I18n.messages` race under differing locales

**File:** `src/main/kotlin/org/bsplines/ltexls/tools/I18n.kt` (around line 19). `object I18n` holds `var messages: ResourceBundle? = null`, mutated by `setLocale(locale)` which is called from `LtexLanguageServer.initialize()` when the client supplies a locale.

**Reproduce:** start the daemon, connect two LSP clients simultaneously, one sending `"locale": "en-US"` in `initialize` params, the other `"locale": "de-DE"`. Trigger something that causes an i18n-formatted log message (e.g. a syntax error in user settings so `I18n.format(...)` runs on both sessions). Observe whether the wrong-language string leaks across.

**Done:** either confirm empirically that LT messages are already effectively English-only (low user impact — leave as known issue) or move the locale state to a `ThreadLocal<ResourceBundle>` / per-server `I18n` instance and add a test.

**Sketch:** replace the `object I18n` singleton with an instance held on `LtexLanguageServer` and threaded through callers. Large blast radius — worth doing only if repro confirms visible breakage.

### 2. Global log-level mutation

**File:** `src/main/kotlin/org/bsplines/ltexls/tools/Logging.kt`. `setLogLevel` on a global `java.util.logging.Logger` is called from `SettingsManager.initialize()`. Last writer wins across sessions.

**Reproduce:** two clients, one with `ltex.trace.server = "verbose"` (or whatever sets debug level), one default. Inspect daemon stderr — expect the verbose one to override for everyone.

**Done:** decide whether "log level is a daemon-wide property" is acceptable (document it) or wire per-session log handlers. Acceptable for POC is fine; document the behaviour in `--help` text or docs.

### 3. No graceful shutdown

**Evidence:** during smoke testing, `SIGTERM` did **not** terminate the JVM — had to use `SIGINT`. Threads are daemon-flagged, but there's no shutdown hook that closes `serverSocket` + joins active sessions with a timeout.

**Reproduce:** start daemon, connect a client, `kill -TERM <pid>`, observe JVM stays up.

**Done:** add a JVM shutdown hook in `LtexLanguageServerLauncher.call()`:
- Close `serverSocket` (this unblocks `accept()` with `SocketException`, which the loop already catches and breaks on).
- Join active session threads with a bounded timeout (say 5s). Current threads are daemon-flagged, so if the timeout expires they die with the JVM.

**Verify fix:** `kill -TERM <pid>` while a client is connected → daemon exits within timeout; client sees connection close.

### 4. Unbounded thread growth

**Reproduce:** loop-open N connections (e.g. N=2000) without closing them. JVM thread count climbs linearly. Eventually either `OutOfMemoryError: unable to create native thread` or OS thread limit.

**Done:** cap concurrent sessions. Options:
- `Semaphore(MAX_SESSIONS)` gate around each spawn.
- `Executors.newFixedThreadPool(MAX_SESSIONS)` instead of raw `Thread(...).start()`, plus a queue-overflow policy.

**Tradeoff:** a too-low cap frustrates legitimate users with many editor sessions; too-high defeats the point. Reasonable starting default: 32. Should probably be a CLI option like `--max-sessions`.

### 5. Log-file interleaving

**Reproduce:** start daemon with `--log-file out.log`, connect two clients, have each exchange a few LSP messages. Inspect `out.log` — expect session A's JSON frames interleaved with session B's, unreadable.

**Done:** either (a) accept current behaviour and document, (b) tag each tee'd line with session ID, or (c) one log file per session (`--log-file-pattern` with `${session}` substitution).

(b) is cheapest; (c) is cleanest. For a POC follow-up, (a) + a doc note is acceptable.

### 6. No multi-client integration test

**Nothing** in `src/test/` exercises `--endless` or concurrent connections today. The existing `LtexLanguageServerLauncherTcpSocketTest` tests one single-shot client and uses `PER_CLASS` lifecycle.

**Done:** new test — `ConcurrentSessionDaemonTest` or similar — that:
1. Starts launcher in a thread with `--server-type=tcpSocket --endless --port=<free>`.
2. Opens N (e.g. 3) sockets in parallel.
3. Sends a minimal `initialize` on each, reads reply, asserts all complete within a timeout.
4. `AfterAll`: close sockets, interrupt launcher thread.

Reuse the polling/connect pattern from the existing `TcpSocketTest` `setUp()` (see lines 62-71 of `LtexLanguageServerLauncherTcpSocketTest.kt`) — Windows runners are slow to bind and a naïve `Socket(host, port)` right after thread start flakes intermittently (issue #151 fix).

### 7. End-to-end LSP flow not exercised

The smoke test only verified *TCP-accept* concurrency, not that the LSP layer actually serves real requests correctly in parallel. The POC commit is defensible because `launch()` creates a fresh `LtexLanguageServer` per call and all known instance-scoped state is isolated — but we have not run a concurrent `initialize → didOpen(markdown) → wait for diagnostics` flow.

**Done:** either #6 above (test harness) or a manual two-client run with a real LSP probe. Confirm each client gets its own diagnostics for its own document and neither client's diagnostics leak to the other.

---

## Design decisions still open

These need a human call before this work can merge upstream.

- **Opt-in gating.** POC changes `--endless` semantics in place: for `tcpSocket`, it is now concurrent. No new flag. Alternative: add `--concurrent` and keep `--endless` serial as the baseline. Maintainers (Daniel) may prefer the latter for merge safety. Decide before PR.
- **CLI help text.** `--endless` still says `"Keep server alive when client terminates."` — now misleading for TCP. Update to mention concurrent behaviour.
- **Relationship to `feat/workspace` PR.** The workspace-folders capability PR on branch `feat/workspace` (commit `63060ec`) is a separate merge. No code conflict expected — they touch different files. But logically this branch depends on workspace-folders being advertised, because the motivation for concurrent daemon is multi-root clients wanting to share a JVM. Consider landing `feat/workspace` first.
- **I18n keys for new log messages.** Strings like `"Accept loop terminating: ${e.message}"` and `"Session $id crashed: ${e.message}"` are raw English literals. For consistency with the rest of the launcher, they should be routed through `I18n.format(key, ...)` with new keys added to the properties bundles (`src/main/resources/LtexLsBundle_*.properties`). Low-priority polish; acceptable POC.

---

## Key references

- **Implementation plan:** `/Users/andrea/.claude/plans/i-am-in-a-sleepy-lynx.md` — full audit of hazards and design rationale. Read this first before touching the code.
- **Companion workspace-folders branch:** `feat/workspace`, HEAD `63060ec` — advertises `workspace.workspaceFolders` capability; unrelated code-wise but thematically linked.
- **Key files:**
  - `src/main/kotlin/org/bsplines/ltexls/LtexLanguageServerLauncher.kt` — the only file modified in `f2c90e9`.
  - `src/test/kotlin/org/bsplines/ltexls/LtexLanguageServerLauncherTcpSocketTest.kt` — existing TCP test, template for multi-client test.
  - `src/main/kotlin/org/bsplines/ltexls/tools/I18n.kt` — hazard #1.
  - `src/main/kotlin/org/bsplines/ltexls/tools/Logging.kt` — hazard #2.
- **Upstream:** `ltex-plus/ltex-ls-plus`. No prior discussion of concurrent daemon mode in issues or PRs at the time of this commit; file an issue before opening the PR to gauge maintainer appetite.

---

## Quick-start for a returning agent

```bash
# Rebuild:
mvn -B -e package -DskipTests

# Start the concurrent daemon on a test port:
target/appassembler/bin/ltex-ls-plus --server-type=tcpSocket --endless --port=52715 &
DAEMON_PID=$!

# Two-concurrent-connections smoke test:
python3 -c '
import socket, time
s1 = socket.create_connection(("localhost", 52715))
s2 = socket.create_connection(("localhost", 52715))
time.sleep(2)
s1.close(); s2.close()
' &
sleep 1
jstack $(pgrep -f 'java.*ltex.*52715') | grep '"ltex-session-'
# Should see ltex-session-1 AND ltex-session-2 as concurrent daemon threads.

# Teardown:
kill -INT $(pgrep -f 'java.*ltex.*52715')
```

Start with hazard #3 (shutdown hook) — it's the smallest, most self-contained improvement and unblocks cleaner testing for the later hazards.
