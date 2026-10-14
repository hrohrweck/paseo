# Platform Matrix — macOS Option-as-Meta terminal setting (paseo #6651)

QA basis: real Mac hardware ("Horst's MacBook Pro", macOS, `navigator.platform = MacIntel`), German
software input source layout basis — Option+L composes `@`. The ⌥L keystroke was synthesized with
pinned trusted CDP `Input.dispatchKeyEvent` parameters (`key:"@"`, `code:"KeyL"`, vk 76, Alt
modifier, `text:"@"`) that reproduce exactly what macOS Chrome produces for ⌥L under the German
input source. Byte-level payload evidence from `window.__paseoTerminal.onData` in the real web app
(dev daemon in the worktree, real pty, `cat -v` pane).

| Platform | Tested | Evidence / expectation |
| --- | --- | --- |
| Web — macOS (Chrome/Chromium) | **YES — this run** | `red-main-payload.txt` (upstream main: ESC-prefixed `\x1bl`), `on-payload.txt` (branch ON: identical `\x1bl`), `off-payload.txt` (branch OFF: literal `@`, byte `0x40`), `toggle-off-recording.mp4`, `settings-row-before.png` / `settings-row-after.png`, `pane-on.png` / `pane-off.png`, `pane-transcript.json` (xterm buffer: ON `^[l` → OFF `^[l@` in the same pane after a live toggle, no reload) |
| Desktop — macOS (Electron) | Not tested here; same web runtime path | Electron renderer is the Metro `web` platform (`isWeb` true → row visible; runtime identical to the web build verified above). Expected identical behavior; not separately exercised in this run. |
| Desktop — Windows / Linux | NO — expected no-op | Option is mac-only: the runtime only consults `macOptionIsMeta` on the xterm mac branch (`navigator.platform` Mac). On Win/Linux the row still renders (it is gated by `isWeb`, not by platform), but Alt behavior is unchanged by this setting on those platforms (xterm's non-mac Alt handling). Default ON changes nothing anywhere. |
| Web — other OS (Win/Linux browsers) | NO — expected no-op | Same rationale as Desktop Win/Linux: non-mac `navigator.platform` never reaches the mac branch; setting has no effect. |
| iOS / Android (native) | Unaffected by design | Settings row is hidden behind the `isWeb` gate (`terminal-section.tsx` renders the row only `if (isWeb)`); the native terminal webview entry passes no `macOptionIsMeta` and the compiled runtime default (`?? true`) preserves today's native behavior. Verified in code review (todo T9 regen evidence); not user-visible on native. |

## Verdicts (IS-1 / IS-2 / IS-3)

- **IS-1 (default unchanged):** upstream main payload `"[l"` (bytes `1b 6c`) === branch-ON payload
  `"[l"` — ESC-prefixed Meta sequence, byte-identical. PASS
- **IS-2 (opt-out types the composed character):** branch-OFF payload `"@"` (byte `40`), single
  character, no ESC. PASS
- **IS-3 (live toggle, no reload):** in one continuous page session the switch was clicked OFF
  (persisted setting `true → false`, runtime `window.__paseoTerminal.options.macOptionIsMeta`
  `true → false`) and the very next dispatch — without any document reload — produced `@` where the
  previous dispatch in the same session had produced `[l`. PASS

## Environmental notes

- The German-layout composition (⌥L → `@`) was reproduced via pinned trusted CDP key events rather
  than a physical keyboard; this exercises the identical Chromium input pipeline (trusted keydown
  with `text` leg) that a German software input source drives. Hardware AZERTY etc. use the same
  composition mechanism.
- ESC-prefixed payload is `\x1b` + `l` (Meta+l): with `macOptionIsMeta` on, xterm maps Alt+KeyL via
  the key code, which is precisely why German users cannot type `@` — the composed `@` never rides
  the text leg.
- Focus-reporting sequences (`\x1b[I`) from terminal re-focus can appear in pane screenshots as
  `^[I`; the byte-level payloads and the xterm buffer transcript are the authoritative evidence.
