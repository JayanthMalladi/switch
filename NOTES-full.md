# Signal for Switch (#792) — the whole runbook, start to finish

One document: environment → the adapter (already built) → live recording
(scenes 0–6 + extras) → submission gates → PR + video + CLA → green CI.
Read top to bottom; every step ends with its gate. Supersedes
`NOTES-scene4.md` and `NOTES-submission.md`.

**Status legend:** `[done]` already finished on this machine · `[you]` a click
or a GitHub action only you do · `[me]` I run it in the terminal and show you
the output · `[gate]` do not continue until it passes.

**Hard rules for this whole file** (from the kickoff; they also keep the PR
safe):
- No real phone numbers, UUIDs or group ids in any file, fixture, commit
  message or PR text. Placeholders: `+<BOT>`, `+<YOU>`, `g:<group>`.
- I never push to `upstream` (`sandbox-quantum/switch`). `origin` is
  `JayanthMalladi/switch`. I never comment on the two competing PRs.
- The PR diff must contain only: `signal/`, the one `PLATFORMS` line,
  `docs/messaging-platforms/signal.md`, `test_signal_*.py`,
  `signal_fixtures/`, and the one line in `test_channel_retirement.py`.

---

## Part A — Environment (setup steps 1–10)

All `[done]` on this box today. Re-check only the marked gates before the
recording day.

1. `[done]` WSL2 Ubuntu + Docker Desktop (WSL2 backend, integration on).
2. `[done]` Toolchain: `git`, `curl`, `build-essential`, `qrencode`,
   `uv`, `just`.
3. `[done]` Fork `sandbox-quantum/switch` → `JayanthMalladi/switch`,
   cloned to `~/switch`, `upstream` remote added,
   `git checkout -b feat/signal-platform`, `just init-env && uv sync && just up`.
4. `[done]` Claude Code installed, pointed at **your inference gateway** via
   `~/.bashrc` (`ANTHROPIC_BASE_URL` / `ANTHROPIC_AUTH_TOKEN` — never
   committed anywhere).
   `[gate]` re-check on recording day:
   `source ~/.bashrc && claude --model <gateway Opus id> -p "say ok"`
   answers. If not, stop — scenes 4–6 depend on it.
5. `[done]` Karpathy guidelines skill installed (`/plugin list` shows
   `karpathy-skills`); `.claude/` kept out of the PR via `.git/info/exclude`.
6. `[done]` `/code-review high` + `/security-review` available (built in;
   run them in Part D).
7. `[done]` signal-cli 0.14.x Linux native build on PATH.
   `[gate]` `signal-cli --version` → 0.14.x.
8. `[done]` Bot number arranged (registered second number, or linked spare
   device), profile name `Switch`.
9. `[done]` Daemon:
   `signal-cli -a +<BOT> daemon --http 127.0.0.1:8080 --receive-mode on-start`
   `[gate]` `curl -s localhost:8080/api/v1/check` → 200.
10. `[done]` Switch Console (early access) on the Windows side, connected to
    `http://localhost:8000` (WSL2 forwards the port).

`[gate]` The original environment proof — run it first on any fresh day:
```bash
just test -k test_platform_self_contained   # real Postgres, no mocks
```
`[done]` passed on this machine; re-run it the morning of the recording.

## Part B — The adapter (build steps 1–10: already committed)

Ten build steps, each committed green in order, 14 commits on the branch
(`git log main..HEAD`). No need to rebuild — this section is the reference
map of *what each commit proves*, which is what reviewers will ask.

| Build step (plan §172) | Commits | The check that closed it |
|---|---|---|
| 1. Live probe of signal-cli, fixtures captured & scrubbed | `test(signal): add scrubbed signal-cli 0.14.9 fixtures from a live probe` | every RPC + 5 SSE events captured from the real daemon; fields in the code come from `signal_fixtures/`, never memory |
| 2. Read the references (guide, `dummy_platform`, `adapter.py` abstract methods, Telegram patterns) | (research, no commit) | abstract-method list built from the branch, not the stale grep |
| 3. Skeleton + registration | `feat(signal): skeleton adapter and registration` | `just typecheck` + `test_bridge_type_registry` / `test_platform_self_contained` green |
| 4. Config, `verify_credentials`, `outbound_urls` | `feat(signal): config validation and credential check` | a wrong URL and a malformed number refused with clear messages |
| 5. JSON-RPC client over `httpx` | `feat(signal): JSON-RPC client and corrected credential check` | transport-level tests (`httpx.MockTransport`) green |
| 6. Outbound (render/styles/UTF-16, chunk, send/edit/delete/typing/attachments) | `feat(signal): outbound …`, `feat(signal): chunk long replies …` | outbound tests green **and** a manual `send` hit the real phone |
| 7. Inbound SSE (lease-only, replay, dedupe, commands, attachments) | `feat(signal): inbound event stream and room relay` | fixture replay green **and** a phone message appeared in a room |
| 8. Failure mapping | `test(signal): map daemon failures to telemetry reasons` | real `httpx` exception classes classify as `network`/credential/operation |
| 9. Docs page | `docs(signal): operator page for connecting Signal` | every config field and every `False` flag explained |
| 10. Full gate | `test(signal): pin card-answer + chunked-send semantics`, `test(signal): pin the edit-refusal seam …`, `test(signal): two pods on real Postgres` | `just check && just typecheck && just test` green |

Design in one paragraph (this becomes the PR design note): the adapter keeps
**zero in-process state**. `channel_id` is `g:<base64 groupId>` /
`d:<uuid>`; `message_ref` is `<author>:<signal-timestamp-millis>`; a
quote-reply is stored as that root's ref, so any pod decodes
`quoteAuthor`+`quoteTimestamp` from it — no quote cache. Only the lease
holder opens the SSE stream (`keeps_quiet_unless_listening`); on reconnect
the daemon replays up to 1000 kept events and the ref-dedupe absorbs the
overlap, so restarts and pod handovers lose nothing and double nothing. A
replay backlog older than the relay window is dropped, so a handover never
re-posts days-old messages or re-runs an old `!command`.

`[gate]` before Part C: `just check && just typecheck && just test` all
green on the current branch tip (re-run: the tree may have moved).

## Part C — The live recording (scenes 0–6 + extras)

Script, captions and target length (2:30–3:00, 1080p, captions on screen,
no voice-over, phone next to Console via scrcpy) live in `NOTES-video.md`.
This section is the **operational** detail for every scene, in order.

### Scene 0 — pre-flight (before pressing record)

All true on this machine as of today:

- [x] switch-core from the branch on `:8000`; `GET /gateway/collaborations/types`
      returns `[…, "signal"]` — Signal is live in the platform list
      (`platforms.py` registration is on the running build).
- [x] Signal bridge row in the DB: `collaboration_bridges` →
      `type='signal'`, `status='active'` (scene 2's result, durable).
- [x] daemon on `:8080` holds exactly one account: `+<BOT>`.
- [x] the "Switch test" Signal group: `+<BOT>` is **admin + member**, your
      phone is a member. (⚠️ A *second* group named "Switch" exists where
      the bot number is only *pending* — do not record in that one.)
- [x] gateway env probe passes (Part A step 4 gate).
- [ ] **the managed agent exists and is the only agent in the group's room**
      (scene 1 of the *next* sub-section, below).

### Making the agent + room exist (one-time; recorded as normal screen time, not a scene)

The key fact (why you never "point your Claude session" anywhere): the
Claude Code session that answers on the phone is **spawned by Console**. You
create a *managed agent* whose provider is Claude Code; when it is addressed
in a room with no live session, Console's embedded controller auto-starts a
session, injects the Switch room tools, and its LLM is your gateway.
("Adding an agent: `server-agents-view.tsx` — *Add one, set how it is
addressed, and start sessions*." `known_agents.py`: an addressed agent with
no live session auto-starts one.)

**Step 1 — create the agent (`[you]`, Console):**
1. Console → your server → **Agents** area → **Add agent**.
2. Provider **Claude Code**, run location **this machine**.
3. Working dir = your `~/switch` checkout (the branch). Name = `switch-dev`
   (short; it bold-prefixes every message and is the word you `@`).
4. **Addressing policy: leave the default** (`addressing_policy` NULL =
   open to the room, `delivery/addressing.py:390`). This is what lets your
   *unlinked* phone answer the card in scene 6. Do **not** set it
   owner-scoped for the recording.
5. Save. It registers as a `clients`/`agents` row in switch-core.

**Step 2 — mint the room (`[you]`, phone):** in the "Switch test" group,
send `hi`. Signal has `supports_channel_creation: False`
(`signal/adapter.py:130`), so the Console's New-Room dialog (which lists
only `canCreateChannels` bridges, `CreateRoomModal.tsx:113-115`) will never
offer Signal — the room for the group is **created lazily** by that first
inbound message (`_create_room_for_channel`, `collaboration_core.py:1116`).
The room now appears in the Console sidebar; note its name.

**Step 3 — add the agent to that room (`[you]`, Console):** open the room →
**Add agents** (`AddAgentsToRoomModal`, `POST /rooms/{room_id}/agents`) →
select `switch-dev`. The room must hold **exactly one agent** — the rule
that makes the `@` in scene 5 unambiguous
(`_resolve_self_mention_target`: a bot mention resolves to the sole agent
when there is one, else it's ambiguous → guidance, not a reply).

`[me] [gate]` the four DB checks (tables verified in `db/models.py`:
`agents` L841, `rooms` L1099, `client_rooms` L805, `agent_sessions` L2239):
```sql
SELECT id, name, addressing_policy FROM agents;                      -- 1 row, policy NULL
SELECT id, name, bridge_type, external_channel_id
FROM rooms WHERE bridge_type = 'signal';                              -- the group's room
SELECT cr.client_id FROM client_rooms cr
JOIN clients c ON c.id = cr.client_id
WHERE cr.room_id = <room>;                                            -- humans + exactly 1 agent
SELECT agent_id, status FROM agent_sessions;                          -- up after first @
```
I run all four in one command and show you the output **before** you record.

### The six scenes, in recording order

**Scene 1 — the platform in the list** `[done]` — Console →
"Connect a messaging app" → hover **Signal** (its name + icon, icon ships
as `signal/icon.svg`). Caption: *"1. Signal in Connect a messaging app,
with its own name and icon."* Proof: the live server's platform list
includes `signal` (checked against `:8000` today).

**Scene 2 — connect** `[done]` — fill `daemon_url` = `http://localhost:8080`
(⚠️ `localhost`, not `127.0.0.1` — the server's outbound policy
(`.env` `OUTBOUND_ALLOWED_PRIVATE_HOSTS`) allows the hostname form and
refuses the bare IP), `account` = `+<BOT>`. **Wrong-number beat first:**
type an E.164-valid number the daemon does not hold → the error names the
accounts the daemon *does* register
(`signal/adapter.py` `verify_credentials`), then connect for real. The
`active` row in the DB is the check. Caption: *"2. Connect: the daemon and
account are checked before saving."*

**Scene 3 — webhooks** `[done, skipped]` — 1 s, optional:
*"3. No webhook: Switch dials out to the daemon."* (Mark **n/a** in the PR.)

**Scene 4 — agent → phone** (the core scene; the one your question was
about): in the Console chat view of the room, ask the agent to post:
> post to the room: **ship it** with `git commit` and 🚀

On the phone: the message arrives with **`switch-dev` in bold** at the head
(the `**name**` marker → BOLD style range, `signal/adapter.py::_attribute`),
`git commit` monospaced, emoji intact (Markdown → Signal `textStyle`
UTF-16 ranges). Caption: *"4. Agent to Signal: name prefix, Markdown as
Signal text styles."*

**Scene 5 — phone → room → agent reply**: phone → "Switch test" →
quote-reply the agent's message: *what's in the repo root?* (the
`@switch-dev what's 2+2?` variant works identically — one-agent room).
It lands in the room as a thread reply; the agent's answer arrives as a
quote of that root; the phone shows the typing indicator while the agent
works. Caption: *"5. Signal to room: a quote-reply is a thread reply; the
agent's answer quotes the root."*

**Scene 6 — approval card answered from Signal**: ask the agent to run a
shell command that needs approval (e.g. `rm -rf /tmp/switch-scratch`). The
card posts on the phone **as text with its handle** (Signal has no
buttons; approvals use `A…` handles, `session/text.py:16-21`). Read the
handle **off the card**, then type the answer in the group:
`A<handle> yes` (never hard-code the number). The card redraws as
**answered from Signal** (`bridge_answers.py` → `service.py::answer_approval`,
`answered_by` = your phone; the room-open policy is why it works unlinked).
Caption: *"6. Approval card: no buttons, so the card is text and the answer
is typed back with its handle."*

### The extra clips (10–20 s each, after the six)

1. `!help` from the phone.
2. Attachment both ways; an oversized file **refused visibly** (bridge media cap).
3. A long reply split cleanly across posts, bold/code intact
   (chunking at the phone's in-full limit, `send_message`).
4. **Restart safety live**: stop switch-core (Ctrl-C on `just run`) → send
   "sent while Switch was down" from the phone → start it → it lands
   **exactly once**, nothing older repeated (scroll to prove no dupes).
   Caption 2: *"Older than 24 h is skipped, so a reconnect never floods the
   room."*
5. Edit + delete of an agent message mirrored on the phone.

`[gate]` video cut: 2:30–3:00 total, every caption on screen, dead time
removed, no numbers visible (blur/crop). **The .mp4 is never committed.**

## Part D — Submission gates (`[me]`, all must be green)

```bash
just check        # ruff format --check + ruff check
just typecheck    # mypy
just test         # full suite, real Postgres
uv run --project core pytest core/tests/switch_core/bridges/collaboration/test_signal_*.py
                  # copy the final "N passed" line — it goes into the PR body
/code-review high
/security-review
```
Red anywhere → stop; fix, re-run, and only then proceed.

**Two-pod evidence (the differentiator — never cut):**
`test_signal_pods.py` on real Postgres (~8.5 min: two switch-core
processes against one fake daemon; only the lease holder streams, a reply
goes out once, the next holder replays from 0, old backlogs don't relay).
Record it, or honestly say in the PR: *evidence is the test, not a live
second pod.*

## Part E — Push + PR (`[you]` on GitHub; `[me]` drafts everything else)

```bash
git fetch upstream && git rebase upstream/main     # main moves daily
just check && just typecheck                       # quick re-gate
git push origin feat/signal-platform
```
PR: `JayanthMalladi/switch:feat/signal-platform` → `sandbox-quantum/switch:main`.
**Title (exact):** `feat(collaboration): add Signal as a messaging platform`
**Body starts with:** `Closes #792`
Fill the template section `## Adding or changing a messaging platform`, in
this order, splicing `NOTES-pr.md` (the draft — do not start over):

1. `Closes #792` · **Platform:** Signal · **How it connects:** dials out.
2. *What this changes* (the `NOTES-pr.md` paragraph).
3. *Adapter test summary*: the `N passed` line from Part D.
4. *Video*: attach the .mp4; tick the six template boxes with timestamps
   (webhook box = n/a).
5. *The "Done when" table*: every line of the issue's list, marked done or
   explained, each row stating **what the user does instead** (the
   `NOTES-pr.md` 10-row capability table is the source: buttons→text card,
   threads→quote-replies, channel creation off, directory search off,
   `channel_on_delete="kept"`, inbound edits not mirrored, 24 h refusal,
   one number for all agents + bold name prefix, custom URL schemes off,
   attachment cap with in-room note).
6. *Design note, 4–5 lines*: identifier scheme, zero in-process state,
   replay+dedupe restart safety; **neutrally** note that a prior
   submission's concerns (an in-memory quote cache; a `127.0.0.1` daemon
   address) are avoided by this design — without naming that PR.
7. *Several pods*: the four bullets from `NOTES-pr.md`.
8. *Live vs test-only, honestly*: live = scenes 1–6 + restart + the
   two-pod test; test-only = failure-mapping rows, the 24 h refusal,
   attachment cap.
9. *Security*: the daemon is unauthenticated → bind to localhost/private
   net; the address goes through `outbound_urls`; the adapter never logs
   message bodies.

## Part F — CLA + deadlines (`[you]` on the PR)

When the CLA bot comments, reply with the **exact** sentence:
> I have read the CLA Document and I hereby sign the CLA

| When (IST) | Milestone |
|---|---|
| Nov 7, noon | target: PR open, CI running |
| Nov 7, 23:59 | hard deadline: PR open; afterwards, CI fixes / review nits only |

CI on the PR must go green. If behind on the morning: cut the stretch
goals, then the extra clips — **never** the six required scenes or the
tests.

## Day-of checklist (collapse this onto your wall)

```
[ ] 0 just test -k test_platform_self_contained (morning gate)
[ ] 1 agent created + in the group's room (Part C, steps 1–3); the 4 DB checks green
[ ] 2 gateway env probe passes
[ ] 3 scenes 4–6 + 5 extra clips recorded; video cut 2:30–3:00, no numbers visible
[ ] 4 Part D gates green; two-pod test shown (or honestly noted)
[ ] 5 rebase on upstream/main; quick re-gate; push feat/signal-platform to origin
[ ] 6 PR body assembled (Part E); mp4 attached; six boxes ticked
[ ] 7 PR opened fork → main; title + Closes line exact
[ ] 8 CLA reply with the exact sentence
[ ] 9 watch CI; fix red only inside our files
```
