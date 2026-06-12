# garm 🐕

**The hound at the gate: AI-assisted review of AUR recipe changes before they build.**

In Norse myth, Garmr is the hound who guards the gate of Hel and howls when
something tries to pass that shouldn't. This `garm` does the same for your
Arch system: before `yay` builds an AUR package, garm inspects what actually
changed in the package's _recipe_ — the PKGBUILD, `.install` scriptlets, and
everything else in the AUR git repo — and flags anything that looks like
supply-chain tampering.

## Why this exists

In June 2026, the ["Atomic Arch" campaign](https://lists.archlinux.org/archives/list/aur-general@lists.archlinux.org/thread/FGXPCB3ZVCJIV7FX323SBAX2JHYB7ZS4/)
compromised **~432 AUR packages**. The playbook was simple:

1. Adopt orphaned packages through the AUR's normal stewardship process —
   no account hacking required, and the package keeps its name, history,
   votes, and accumulated trust.
2. Push a tiny recipe change that runs
   `npm install atomic-lockfile minimist chalk` during build/install.
3. The malicious npm package's `preinstall` hook executes an ELF credential
   stealer (browser cookies, SSH keys, GitHub/npm/Slack/Discord tokens)
   with an optional eBPF rootkit, Tor C2, and systemd persistence.
   ([analysis](https://ioctl.fail/preliminary-analysis-of-aur-malware/))

The upstream software was never touched. **Only the build recipe changed**,
and each malicious commit was a 2–3 line diff that anyone reviewing would
have caught. The problem isn't that the diffs are unreadable — it's that
nobody reads them. garm reads them.

## Why not a pacman hook?

pacman/alpm hooks fire during the _transaction_ — but by then `makepkg` has
already executed the PKGBUILD's `prepare()`/`build()` functions as your user,
which is exactly when build-time payloads run. The only safe interception
point is **before makepkg**, which means wrapping the AUR helper. garm wraps
`yay`.

## How it works

```
garm
 ├─ pacman -Qm                      → your installed AUR packages
 ├─ git fetch each AUR repo         → has the recipe changed since you last approved it?
 ├─ AUR RPC metadata                → did the maintainer change? orphan re-adoption?
 ├─ static scanner                  → hard IoCs: pipe-to-shell, npm/base64/eval,
 │                                    known campaign indicators (atomic-lockfile, .onion, temp.sh)
 ├─ claude -p (haiku by default)    → AI review of the diff + metadata with a
 │                                    supply-chain rubric → OK / SUSPICIOUS / MALICIOUS
 ├─ verdicts                        → OK auto-approves; SUSPICIOUS prompts you
 │                                    (or auto-blocks with --paranoid); MALICIOUS blocks
 └─ yay -Syu --ignore <blocked>     → normal update for everything that passed
```

garm keeps its own per-package baseline (`approved commit` + `maintainer`) in
`~/.local/state/garm/state.json`. The review trigger is **"the recipe changed
since you last approved it"**, not "there's a version update" — which means it
also catches recipe changes with _no_ version bump, the signature of this
attack. The maintainer-change signal is weighted heavily: in the Atomic Arch
campaign, the maintainer change was the leading indicator, days before any
malicious commit landed.

A hard static-scanner hit elevates the verdict to at least SUSPICIOUS even if
the AI says OK. If the `claude` CLI isn't installed, garm degrades gracefully
to static checks + maintainer-change detection.

## Usage

```sh
garm               # review pending AUR recipe changes, then run `yay -Syu`
garm --check       # review only; don't invoke yay (exit 1 if anything blocked)
garm --init        # baseline all installed AUR packages as trusted, no review
garm -S <pkg>...   # review new package recipes, then `yay -S <pkg>...`
garm --paranoid    # auto-block SUSPICIOUS instead of prompting
garm --tips        # print the full maintenance tips list
```

On first use, verify your system is clean (check your packages against any
published compromise lists, scan `~/.cache/yay` for IoCs), then run
`garm --init` to set the baseline. From then on, `garm` is your update command.

After each successful update, garm prints one rotating tip from a curated
list of Arch maintenance habits — orphan cleanup, `.pacnew` merges, cache
trimming, etc. — each with the exact command, because nobody remembers the
switches. Edit the list at `~/.config/garm/tips.txt`.

## Install

```sh
git clone https://github.com/carloseberhardt/garm
ln -s "$PWD/garm/garm" ~/.local/bin/garm
garm --init
```

Requirements: `python3`, `git`, `pacman`/`yay`, and optionally the
[Claude Code CLI](https://claude.com/claude-code) (`claude`) for the AI
review pass. Each package review is a single small-model call (haiku by
default) — typically a fraction of a cent. Override with
`GARM_MODEL=claude-sonnet-4-6 garm`.

## What garm can't do

- **It vets recipes, not upstreams.** If the upstream project itself is
  compromised, a checksum-pinned `source=()` will faithfully fetch the
  compromised release and no recipe review will catch it. Different threat.
- **The AI verdict is advice, not proof.** It is very good at "npm appeared
  in a Rust tool's PKGBUILD the same day the maintainer changed," but a
  sufficiently subtle backdoor could be judged OK. Read SUSPICIOUS diffs
  yourself: garm keeps the repos under `~/.local/state/garm/repos/`.
- **First installs have no baseline.** garm reviews the full PKGBUILD, but
  you're trusting a snapshot, not a diff history.

Defense in depth still applies: keep your AUR footprint small, prefer
official repos when possible, read the diffs garm shows you, and consider
`npm config set ignore-scripts true` if your workflow tolerates it.

## License

MIT
