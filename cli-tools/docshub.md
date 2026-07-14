# `docshub` — Documentation hub aggregator

`docshub` collects Markdown documentation from the PayOS project modules and assembles a
unified, topic-oriented hub, generating per-section and master indexes. It lives in the
`payos-docshub` module as cross-platform scripts (`docshub.ps1`, `docshub.sh`,
`docshub.cmd`).

> This `payos-docs` project is the **authored, code-grounded** documentation set. `docshub`
> is a complementary mechanism that aggregates Markdown that already lives inside individual
> module repositories into a single browsable hub. Use whichever fits your workflow.

## Usage

```bash
# PowerShell
./docshub.ps1 -Root <payos-root> -Output <hub-dir>

# Bash
./docshub.sh --root <payos-root> --output <hub-dir>
```

### Parameters

| Parameter | Alias | Default | Purpose |
| --- | --- | --- | --- |
| `--root` | `-r` | current directory | Root directory containing the PayOS module folders. |
| `--output` | `-o` | `<root>/docs-hub` | Destination directory for the assembled hub. |
| `--dry-run` | | off | Preview operations without writing files. |
| `--force` | `-f` (PowerShell only) | off | Overwrite existing destination files (including read-only). |
| `--verbose` | `-v` (bash) / `-Verbose` (PowerShell) | off | Verbose logging. |

The PowerShell entry point also accepts GNU-style long options (`--root`, `--output`,
`--dry-run`, `--force`).

> `-f` is a short alias for `--force` only in `docshub.ps1` (its GNU-style option parser
> recognizes `--force`, `-f`, and bare `force`). The bash script `docshub.sh` only accepts the
> long `--force` form — there is no `-f` short alias in bash. The bash script does define
> `-v`/`--verbose` as an alias pair, unlike PowerShell's `-Verbose`-only switch.

## Examples

```powershell
# aggregate everything under the PayOS workspace into ./hub
./docshub.ps1 -Root "C:\Projets\DTS\MedTech@Work\Innovation\PayOS" -Output "C:\docs\payos-hub"

# preview without writing
./docshub.ps1 -Root . -Output ./hub -DryRun
```

## What it produces

- A topic-oriented tree under `--output`.
- Auto-generated `README.md` indexes per section.
- A master `README.md` linking the sections.

## When to use which

| Goal | Use |
| --- | --- |
| Read the authoritative, curated, code-grounded docs | this `payos-docs` project |
| Snapshot/aggregate per-module Markdown across the workspace | `docshub` |

## Next

- [Documentation home](../README.md)
