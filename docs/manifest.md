# Manifest reference

By default `cmt` reads `cmt.hcl` in the current directory (override with
`-f`). The manifest is [HCL2](https://github.com/hashicorp/hcl).

```hcl
env {
  NAME  = "api"
  IMAGE = "example/api"
}

hosts_group "production" {
  hosts = ["api1.example.com", "api2.example.com"]
  env = { DEPLOY_ENV = "prod" }
}

hosts_group "staging" {
  inventory = "curl http://example.com/latest/meta-data/hostname"
}

command "restart" {
  desc   = "Restart example container"
  run    = "sudo docker restart example"
  serial = 2
}

command "build" {
  desc = "Build image"
  run  = "sudo docker build -t image:latest ."
  once = true
}

command "prepare" {
  desc  = "Prepare to upload"
  local = "npm run build"
}

command "upload" {
  desc = "Upload dist files"
  upload {
    src = "./dist"
    dst = "/tmp/"
  }
}

target "deploy" {
  commands = ["build", "prepare", "upload", "restart"]
}
```

## Blocks

### `env { ... }`

Global environment variables, exported (as `KEY='VAL'` shell-quoted
assignments prefixed onto the command line) for every command. A
`hosts_group`'s own `env` overrides these for its hosts; `-e`/`--env` on the
CLI overrides both.

!!! warning "Shell expansion timing"
    Because these are a *shell prefix* (`NAME='api' ... your-command`, all one
    line), they only reach a **child process's** environment — ordinary POSIX
    shell semantics, not a `cmt` quirk. `run = "echo hello $NAME"` prints
    nothing: `$NAME` is expanded while the outer shell parses the line,
    before the assignment takes effect. To use a value inside the same
    command line, defer expansion to a nested shell:
    `run = "sh -c 'echo hello $NAME'"` (note the single quotes, so the
    *outer* parse leaves `$NAME` untouched). A script file that reads `$NAME`
    normally, invoked via `run = "./deploy.sh"`, is unaffected — this only
    bites literal variable references written directly on the
    `run =`/`local =` line.

### `hosts_group "name" { ... }`

A named set of hosts. Either:

- `hosts = [...]` — a static list. Each entry is `host`, `user@host`, or
  `user@host:port`. The literal hosts `localhost`, `127.0.0.1`, and `::1`
  always resolve to a local (no-network) connection, regardless of any
  `ssh`/`winrm`/`become` block.
- `inventory = "shell command"` — a dynamic list: the command runs locally
  and its stdout, one host per non-empty line, becomes the host list.

`hosts` and `inventory` are mutually exclusive; exactly one is required.

Optional nested blocks configure how `cmt` connects to this group's hosts
(all fields optional; see
[`manifest/manifest.go`](https://github.com/configuration-management-tool/cmt/blob/main/manifest/manifest.go)
for the authoritative field list):

- `ssh { user, port, password, private-key, passphrase, host-key-check,
  known-hosts, tty, tmpdir, connect-timeout }` — SSH is the default transport
  for any non-local host, even with no `ssh` block at all.
- `winrm { user, port, password, ssl, ssl-verify, ca-cert, transport,
  client-cert, client-key, connect-timeout, tmpdir, path }` — used only when
  a `winrm` block is present (a deliberate value-add over sup, which has no
  Windows transport; free here since `go-remoteexec/transport` already
  speaks WS-Management).
- `become { method, user, password }` — privilege escalation (`method` is
  `sudo` (default), `su`, or `doas`).

### `command "name" { ... }`

Exactly one of:

- `run = "shell command"` — runs on every resolved host of the
  `hosts_group` the command is invoked against.
- `local = "shell command"` — runs once, locally, regardless of the
  `hosts_group` (sup's `local` semantics — this is *not* per host).
- `upload { src, dst, executable? }` — copies `src` to `dst` on every
  resolved host (`executable` chmods the remote file `+x`; POSIX hosts only,
  ignored under WinRM).

Optional: `desc` (documentation only), `serial = N` (cap concurrent hosts;
0/unset = unlimited fan-out), `once = true` (run on only the first resolved
host — `serial`/`once` apply only to `run`/`upload`, not `local`).

### `target "name" { commands = [...] }`

A named, ordered list of command names. Commands run in sequence; if any
host fails a command (a transport error, or a nonzero exit code for
`run`/`local`), the remaining commands in the target do not run (fail-fast,
matching sup).

## Supported / deferred

**Supported:**

- Hosts groups: static `hosts` lists and dynamic `inventory` commands.
- Global and per-group environment variables, plus `-e`/`--env` CLI
  overrides; builtin `$CMT_HOSTS_GROUP`/`$CMT_HOST`/`$CMT_USER` variables.
- Commands: `run` (remote), `local` (runs once, not per host), `upload`.
- `serial` (bounded concurrency) and `once` (first-host-only).
- Targets: named, ordered, fail-fast command sequences.
- Host filtering via `--only`/`--except`.
- Piped, non-interactive stdin forwarded to every host's `run` command.
- SSH (default transport) and WinRM (opt-in), plus `sudo`/`su`/`doas`
  privilege escalation, all via `go-remoteexec/transport`.

**Deferred, not stubbed:** a true *interactive* `stdin` session — sup's
`stdin: true` live, keystroke-by-keystroke, multiplexed-output session across
several hosts at once. `remoteexec.Connection.Exec` returns one buffered
`Result` after a command finishes; it does not stream stdout/stderr live, so
a real interactive multi-host session would need a streaming extension this
project does not add. Piped (fully-buffered, known-in-advance) stdin works
today; an interactive terminal session does not.
