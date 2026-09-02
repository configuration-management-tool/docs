# configuration-management-tool

`cmt`: a super-simple deployment tool. It runs named shell commands, defined
in a manifest, against named groups of hosts in parallel.

Part of the **Configuration management** family of the
[pure-Go ecosystem](https://go-desktop.github.io/) — 1 module,
`CGO_ENABLED=0`.

## What is here

This site is the organisation's reference index: what the module is, how its
manifest and CLI work, and where its API documentation lives. The API itself
is generated from the source and served by [pkg.go.dev](https://pkg.go.dev/),
which is always current with the released tags — duplicating it here would
only create a second copy to go stale.

- **[Modules](modules.md)** — every module in configuration-management-tool,
  with its source and its reference.
- **[Manifest reference](manifest.md)** — the HCL2 blocks `cmt` reads:
  `env`, `hosts_group`, `command`, `target`.
- **[CLI usage](cli.md)** — flags, targets, host filtering, piped stdin.

## What `cmt` is

`cmt` is a from-scratch, pure-Go reimplementation of the idea behind
[pressly/sup](https://github.com/pressly/sup) ("Stack Up") — full credit to
that project for the design this borrows — but `cmt` is **not**
Supfile-compatible. The manifest language is
[HCL2](https://github.com/hashicorp/hcl) instead of YAML, a group of hosts is
called a `hosts_group` instead of a `network` (to avoid the networking
connotation), and connections (SSH, WinRM, local exec, privilege escalation)
are handled by the shared library
[`go-remoteexec/transport`](https://github.com/go-remoteexec/transport)
instead of a bespoke implementation.

## What every module here is held to

- `CGO_ENABLED=0`: no cgo, and no shelling out to a command-line tool in
  place of a library — except where a manifest itself is explicitly asking to
  run a shell command, which is the whole point of a deploy tool.
- 100% statement coverage as a CI gate, error branches included.
- BSD-3-Clause.

The standard is described in full on the
[ecosystem map](https://go-desktop.github.io/docs/latest/standards/).
