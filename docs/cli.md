# CLI usage

```
cmt [OPTIONS] HOSTS_GROUP COMMAND [COMMAND2 ...]
```

A `COMMAND` that names a `target` expands to that target's command list.

| Flag | Description |
|---|---|
| `-f FILE` | manifest path (default `cmt.hcl`) |
| `-e`, `--env KEY=VAL` | set/override a manifest env var (repeatable) |
| `--only REGEXP` | keep only resolved hosts matching `REGEXP` |
| `--except REGEXP` | drop resolved hosts matching `REGEXP` |
| `-D`, `--debug` | print a per-host result summary after each command |
| `--disable-prefix` | don't prefix output lines with `[host] ` |
| `--version` | print the version and exit |
| `-h`, `--help` | print usage and exit |

```sh
cmt production build prepare upload restart
cmt production deploy                        # target expands to its commands
cmt production restart --only 'api1\.'        # only api1.example.com
echo 'some input' | cmt production restart    # piped stdin reaches every host's command
```

## Piped stdin

When `cmt`'s own stdin is not a terminal, it is read fully and replayed (a
fresh copy per host) as the standard input of every `run` command's remote
execution — matching `echo cmd | sup group cmd` in upstream sup.

## Development

```sh
go build ./...
go vet ./...
go test -race ./...
```

`manifest` and `connect` are pure unit tests (HCL parsing; host-string and
SSH/WinRM/`become` config-shape mapping) plus one real, non-mocked
integration test each through `transport.NewLocal()` — no network, no
in-process SSH/WinRM server (that protocol is already tested inside
`go-remoteexec/transport` itself). `orchestrate` is tested against a fake
in-memory `transport.Connection`, exercising `serial` batching, `once`,
target fail-fast, env layering, and host filtering without any real
transport. `cmd/cmt` covers flag parsing and runs one real end-to-end
`local` command through the actual CLI entrypoint.
