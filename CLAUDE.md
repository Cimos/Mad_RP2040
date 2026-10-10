# Mad_RP2040 — project & CI guide

RP2040-based test PCB for a larger keyboard project. KiCad 8 design. The PCB,
schematic, and custom design rules (`Mad_RP2040.kicad_dru`, JLCPCB/PCBWay) live
at the repo root; `mad_lib/` is a git submodule (symbols/footprints/3D models).

## Working conventions

- **Always merge via PR to `main`** (squash). Don't push to `main` directly.
- After cloning for CI work, fetch submodules: `git clone --recurse-submodules`
  or `git submodule update --init --recursive` (CI uses `submodules: recursive`).
- Workflows live in `.github/workflows/`. All CI jobs are least-privilege
  (`permissions: contents: read`) except the comment jobs noted below.

## CI/CD overview

All board generation is delegated to the **`Cimos/kibot-config`** GitHub Action
(a KiBot-in-Docker datapack builder). Each workflow checks out that repo for its
config YAMLs *and* invokes it as the action.

| Workflow | Trigger | Purpose |
|---|---|---|
| `build-datapacks.yaml` | PR + push `main` | PCB / 3D / 2D-image datapacks (matrix) + panel (job) |
| `build-diff-action.yaml` | PR | PCB visual diff vs base |
| `build-video-action.yaml` | push `main` | render rotating-PCB frames → stitch MP4 (ffmpeg) |
| `release.yaml` | tag `v*` | build datapacks → zip → **publish GitHub Release** |
| `design-review.yaml` | PR | **warn-only ERC/DRC** gate → PR comment |
| `kicad-happy.yml` + `kicad-happy-comment.yml` | PR (+ `workflow_run`) | **AI design review** (deterministic) → PR comment |

Triggers are scoped with `paths:` (KiCad/lib/config files only) and
`concurrency:` cancels superseded runs. Builds run on PRs + `main`, not on every
branch push.

## Cutting a release

```sh
git tag vX.Y.Z && git push origin vX.Y.Z
```

`release.yaml` rebuilds from the tagged commit and publishes a Release with
`Mad_RP2040-vX.Y.Z-{pcb-datapack,3d-model,2d-images,panel-datapack}.zip` +
auto-generated notes. Restricted to `v*` tags so other tags don't publish.

## Action pinning policy

- **CI workflows track `Cimos/kibot-config@main`** (latest features).
- **`release.yaml` SHA-pins `kibot-config`** (currently `b6991f9`) so a re-run of
  an old tag rebuilds byte-identically. `uses:` can't take an env var, so the SHA
  is hardcoded per step — bump it deliberately to adopt upstream changes.
- **`kicad-happy` and `thollander/actions-comment-pull-request` are SHA-pinned**
  (third-party, see security note). Re-pin only after re-auditing.

## Design-review gates (both warn-only, complementary)

1. **ERC/DRC** (`design-review.yaml`): re-enables ERC + DRC (disabled in
   `kibot-config/options.yaml`) via the project-local **`design-review.kibot.yaml`**,
   using `Mad_RP2040.kicad_dru`. Posts/updates one PR comment (marker
   `<!-- kibot-design-review -->`) via `gh` CLI — **no third-party action**.
   Job-split: the third-party action runs `contents: read`; only the separate
   comment job holds `pull-requests: write`.
2. **kicad-happy** (`aklofas/kicad-happy`): semantic/AI-style checks (power tree,
   derating, protocol, SPICE, EMC, DFM). **Deterministic tier only — AI is OFF.**
   Uses the **fork-safe two-workflow pattern**: analysis runs on `pull_request`
   (read-only token) and uploads a report artifact; `kicad-happy-comment.yml`
   posts it via `workflow_run` (trusted, `pull-requests: write`). The comment half
   only fires from the **default branch**.

### kicad-happy security (audited — keep these invariants)

Audited at **v1.3.2 / `839d9b03c42358ab16f2eedfdea6c4bc6469826f`**: no design-file
exfiltration, no remote code exec, only `statuses: write` for itself. Verdict:
GO-WITH-MITIGATIONS. **Keep:** SHA pins (kicad-happy + thollander); fork-safe
pattern (never `pull_request_target`); **no distributor API keys** (avoids MPN
egress); **no `claude-code-action`/`ANTHROPIC_API_KEY`** (that's the only path
that sends design data to an LLM — enabling it needs a separate audit).
**Re-audit on every version bump** (single-maintainer project, unsigned tags).

## Known issues / TODO

- **`kbd` footprint library is not vendored.** `fp-lib-table` points the `kbd`
  lib at `E:/git/General/kbd/kicad-footprints/kbd.pretty` — an absolute path that
  only resolves on the original author's machine; those `keyswitch_choc12_*`
  footprints are **not in the repo or `mad_lib`**. The board still builds (KiCad
  embeds footprint geometry in the `.kicad_pcb`), but ERC/DRC emit
  `lib_footprint_issues`/`footprint_link_issues` warnings. **Fix:** vendor
  `kbd.pretty` into the repo (or add as a submodule) and repoint the entry to
  `${KIPRJMOD}/...`.
- **Missing I²C pull-ups** on SDA/SCL — flagged by kicad-happy; verify in schematic.
- **Video MP4 step is unverified in CI** — it can't run on PRs (push-`main`-only)
  and ffmpeg wasn't available to test locally. The logic is standard and
  `continue-on-error`; **check the first push-to-`main` run** that produces frames.
