# Ghostty: Backspace on the last Hangul jamo

A one-patch local build of [Ghostty](https://github.com/ghostty-org/ghostty) 1.4.0 for macOS that fixes
Backspace when only one Hangul jamo is left in the composition.

This is a personal, temporary build. It is **not** an official Ghostty release.

## The bug

With the built-in macOS 2-Set Korean input method on Ghostty 1.4.0:

| Keys | Expected | Ghostty 1.4.0 |
|---|---|---|
| `안` then Backspace x3 | `안` → `아` → `ㅇ` → (empty) | `안` → `아` → `ㅇ` → `ㅇ` |

The last jamo comes back as real text and needs a second Backspace. Upstream tracks this as
[ghostty-org/ghostty#14797](https://github.com/ghostty-org/ghostty/issues/14797) (a regression from 1.3.1).

## Cause

In `macos/Sources/Ghostty/Surface View/SurfaceView_AppKit.swift`, `keyDown`:

1. When Backspace is pressed on a lone jamo, the Korean input method commits the jamo with
   `insertText` and then hands Backspace back to the app so that the app deletes it again.
2. Ghostty sends text committed during a composition to the terminal, and afterwards replays the
   original key only if it is an arrow key (`shouldReplayCommittedPreeditKey`).
3. Backspace is not replayed, so the jamo is written to the terminal and the delete is lost.

The regression and the commit-then-delete behaviour were first identified by other people in the
upstream issue thread and in PR #15048. I read the 1.4.0 source to confirm it and wrote this patch
for my own use.

## The fix

[`patches/0001-macos-korean-last-jamo-backspace.patch`](patches/0001-macos-korean-last-jamo-backspace.patch)

If a plain Backspace makes the input method commit exactly the text that was in the preedit before
the key, nothing is sent: the terminal never had that text, so there is nothing to delete. The
preedit is already cleared at that point.

Not affected: syllable decomposition (`갑` → `가` → `ㄱ`), commits caused by other keys, modified
Backspace, and input methods that do not commit on Backspace.

The patch adds unit tests for the new helper in `SurfaceViewAppKitTests`.

## How it is built

`.github/workflows/build.yml` runs on a GitHub-hosted macOS runner and:

1. checks out the official source at the exact commit of tag `v1.4.0`
   (`83fb95624b0439ee660c2679b43b38c8b63769cd`) and verifies the hash,
2. applies the patch and prints the resulting diff,
3. installs Zig 0.17.0 with a pinned SHA-256,
4. builds with upstream's own `ReleaseLocal` configuration,
5. uploads the app together with `BUILD_INFO.txt` (commit, patch hash, artifact hashes).

No secrets are used and the workflow token is read-only.

## Caveats

- The app is ad-hoc signed. It is not signed with Ghostty's Developer ID and not notarized by Apple.
- Automatic update checks are off in this build. Go back to the official release once the upstream
  fix ships.
