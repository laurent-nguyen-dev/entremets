# entremets

**Track changes in the system components your HPC code depends on, using Git as the history.**

> Status: early design. The specification lives in [`openspec/`](openspec/); no release yet.

## The idea

On HPC clusters, kernels, OS packages, drivers and environment modules are updated
regularly, often without users knowing what changed, when, or on which nodes.
When your code starts crashing or producing different results, you need to know
what moved underneath it.

`entremets` uses a **Git repository as both its configuration and its memory**:

- `main` holds `entremets.toml`, which describes the *objects* to watch
  (a package version, the kernel, a text file, an environment variable,
  a module, the output of any command…), plus your own plugins in `plugins/`.
- Each *context* (login node, Slurm partition) gets an orphan branch
  `state/<context>` with one plain-text state file per object.
- `entremets scan --repo <repo>` scans the current machine in parallel. Every
  object whose state changed becomes **one commit**, tagged with the node and
  context; unchanged objects produce nothing. Changes are pushed automatically,
  and critical changes trigger a summary e-mail.

Your system's history is then just `git log state/gpu`, `git diff state/cpu state/gpu`,
and every Git tool you already know.

Scheduling is left to you: run it from cron on a login node, or at the top of
your batch jobs.

## Planned commands

| Command | Purpose |
|---|---|
| `entremets init --repo <repo>` | Initialise an empty repository with a commented example configuration |
| `entremets check --repo <repo>` | Validate the configuration and plugins |
| `entremets scan --repo <repo>` | Scan the current machine and commit/push changes |
| `entremets plugins --repo <repo> [name]` | List available object types, or show one in detail |
| `entremets objects --repo <repo>` | List configured objects |
| `entremets add --repo <repo> <name> --type <plugin> ...` | Add an object |
| `entremets remove --repo <repo> <name>` | Remove an object |

## Development

This project is developed with [OpenSpec](https://github.com/Fission-AI/OpenSpec):
changes are proposed and specified under `openspec/changes/` before being
implemented. (During the initial phase, the specification documents are written
in French; they will be translated to English before the first public release.)

## License

Licensed under either of

- MIT license ([LICENSE-MIT](LICENSE-MIT))
- CeCILL-B license ([LICENSE-CeCILL-B](LICENSE-CeCILL-B))

at your option.
