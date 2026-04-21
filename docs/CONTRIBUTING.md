# Contributing

Workflow, commit conventions, and chat-sprint handoffs.
Paired with [DEV_ENVIRONMENT.md](DEV_ENVIRONMENT.md) for tooling rules.

---

## Chat-sprint model

Work is organized in numbered "chats" (Chat 01 through 10 so far). Each chat is a scoped feature area, not a literal chat transcript. See [CHANGELOG.md](CHANGELOG.md) and [ROADMAP.md](ROADMAP.md) for the mapping.

When starting a session:
1. Read the last entry of [CHANGELOG.md](CHANGELOG.md).
2. Read the current "Unreleased" section and top of [ROADMAP.md](ROADMAP.md).
3. Pick one of the priority items (Critical → High → Medium → Low).
4. Work in a single focused area; defer unrelated cleanup.

When ending a session:
1. Update [CHANGELOG.md](CHANGELOG.md) — move shipped items from `[Unreleased]` to a new dated section, OR extend the current sprint section.
2. Update [ROADMAP.md](ROADMAP.md) — move completed items, add newly-discovered bugs or ideas.
3. Commit with a clear sprint reference in the message body.

---

## Commit conventions

- **First line:** imperative, ≤72 chars, no trailing period.
  - Good: `Wire traffic relay on gatewayChanged event`
  - Bad: `updated vpn stuff`
- **Blank line, then body** (optional) with:
  - What & why (not how — the diff shows how)
  - Chat reference if applicable: `Part of Chat 10 / OpenWRT package`
  - Known limitations introduced or left open
- **No emoji**, no `[ci skip]` unless actually skipping CI.
- **Never** commit:
  - `.xcframework/` or `ios/Frameworks/*` binaries — rebuild via gomobile
  - `ios/Pods/`, `android/.gradle/`, `node_modules/`, `ios/build/`
  - Secrets, real UUIDs beyond what's already tracked as a security issue

Example:
```
Add internet reachability probe to election score

Nodes now curl the exit server every 30s and multiply their score
by {0, 1}. Prevents captive-portal / DNS-blocked nodes from winning
election and black-holing traffic.

Part of Chat 11 / resilience work. See SECURITY.md issue #6.
```

---

## Branch strategy

Currently: single `main` branch, direct commits.

When opening to external contributors:
- Feature branches: `feat/<short-name>`
- Bug fixes: `fix/<short-name>`
- Docs: `docs/<short-name>`
- PRs require: typecheck pass + manual smoke test on Android + iOS.

---

## Code style

### TypeScript (React Native)
- Functional components + hooks only. No class components.
- Zustand for cross-screen state. Keep component-local state in `useState`.
- No `any` unless talking to a native bridge with loose typing.
- ESLint config in `.eslintrc`; run `npm run lint` before commit.

### Swift (iOS + Mac)
- `async`/`await` for anything involving I/O.
- Mark UI-facing types `@MainActor`.
- Error handling via `throws` — not returning optional errors.
- No force unwraps (`!`) in shipped code; `guard let` or `if let`.

### Kotlin (Android)
- Coroutines + `Flow` for async.
- Use `sealed class` for state enums.
- Prefer `val` over `var`.
- Inject via constructors; no singletons except `NeighborRepositoryHolder` (intentional).

### Comments
- Write none by default. Names explain what; only comment **why** when the reason is non-obvious (a workaround, a subtle invariant, a past bug this guards against).
- Never comment "used by X" — that rots.

---

## Testing

Currently manual. Priorities for automated testing (none yet):

1. **TypeScript typecheck** on every commit (fast, catches lots).
2. **Android unit tests** for `MeshController` election logic (pure Kotlin, fast).
3. **Swift unit tests** for `MeshDiscovery` (pure Swift, fast).
4. **Integration test** harness: three virtual nodes running in Docker, verify election.

No UI tests planned — the surface is small and they're expensive.

---

## Pull request checklist

(When PR workflow turns on)

- [ ] Title references the chat sprint or roadmap item
- [ ] `npm run lint` passes
- [ ] `tsc --noEmit` passes
- [ ] Tested on one physical Android device
- [ ] Tested on one physical iOS device
- [ ] Mac app tested if changes affect `~/OpenBandMac/`
- [ ] [CHANGELOG.md](CHANGELOG.md) updated under `[Unreleased]`
- [ ] [ROADMAP.md](ROADMAP.md) updated if priorities shifted
- [ ] No new hardcoded secrets (grep for `UUID`, `<exit-server-ip>`, hotkeys)
- [ ] No new binaries committed

---

## Reporting bugs

For now, via direct message. When we open up:

Create an issue with:
1. **Platform + version** — iOS 17.3.1, Android 14, macOS 14.x
2. **Device model + identifier** (see [DEV_ENVIRONMENT.md](DEV_ENVIRONMENT.md))
3. **Steps to reproduce**
4. **Expected vs actual behavior**
5. **Logs** — Xcode console for iOS, `adb logcat` output for Android, copy-pasted
6. **Mesh topology** — how many nodes, same WiFi or different, Mac plugged in or not

Security bugs: email `soroushosivand@gmail.com` privately, NOT a public issue. See [SECURITY.md § Reporting](SECURITY.md#reporting).
