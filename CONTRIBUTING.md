# Contributing

This guide explains how the sandbox service works inside, which rules its code follows, and how to check and submit a change. People and coding agents follow it alike, and [AGENTS.md](https://github.com/davlgd/biscuit-sandbox/blob/main/AGENTS.md) adds what agents check before they report a change as done.

## Architecture

The service is an application built on biscuit-ssh. It mints tokens with `biscuit-ssh-core`, and serves its API with the plumbing of `biscuit-ssh-service`. Both crates come from crates.io, or from a checkout of the biscuit-ssh repository in `../biscuit-ssh` while developing, as `Cargo.toml` declares. The bastion of biscuit-ssh, with its `[containers]` section, opens the SSH sessions.

| Path               | Role                                                             |
| ------------------ | ---------------------------------------------------------------- |
| `src/api.rs`       | The REST API, its authentication by address, and its JSON errors |
| `src/config.rs`    | The configuration file                                           |
| `src/docker.rs`    | Docker operations on a sandbox: pull, run, check, and remove     |
| `src/image.rs`     | Docker Hub image references, checked before Docker sees them     |
| `src/lifecycle.rs` | Creation, renewal, deletion, recovery at startup, and the reaper |
| `src/main.rs`      | Startup, and the loop that runs the reaper                       |
| `src/state.rs`     | The persistent state and its projection, the revocation list     |
| `tests/e2e.sh`     | End-to-end scenarios against a running service and bastion       |

### State

`state.json` is the single source of truth. The service writes it atomically before any effect that relies on it, then rewrites the revocation list, its projection, after each change. A file lock keeps one instance per state directory.

A sandbox goes from `Creating`, persisted before the pull, to `Active`, then `Deleting`. At startup, the service undoes `Creating` entries, and completes `Deleting` ones.

### Reaper

The reaper removes expired sandboxes, and the containers of this instance that the state doesn't know. When the container of a sandbox vanished, it marks the sandbox `Deleting`, then forgets it on the next pass. The service acts on containers by ID, and only looks names up with the label of the instance.

The reaper acts on Docker listings that may predate the state. It checks a sandbox missing from a listing again, by ID, before it deletes it, and each pass rewrites the state and its projection. The tests of `lifecycle.rs` hold both rules against a fake Docker socket.

### Tokens

Biscuit expirations are inclusive, so a token expires at `expires_at - 1`, and the reaper removes the container from `expires_at`. Tokens only appear in creation and renewal responses, never in logs or listings. A renewal revokes the previous token, and must extend the sandbox's life by at least 30 seconds to bound the revocations a client causes.

### API

Every request from an address not allowed gets the same `401`, before the service reads its body. A creation runs in a task of its own, so a client that disconnects can't leave a reservation behind. Every error is JSON, including the errors of the framework, which the authentication middleware rewrites. Errors never carry the messages of Docker.

## Code rules

The code follows the rules of biscuit-ssh, which its CONTRIBUTING.md details:

- Rust edition 2024, stable toolchain, and the latest released version of each new crate
- Fail closed, and no `unwrap`, `expect`, or `panic!` outside tests or justified cases
- Never log tokens or private keys
- A doc comment on every public item, and comments that explain why, not what

Update the documentation in the same change as the code. The README covers usage and deployment, and this guide covers development. Record user-facing changes in [CHANGELOG.md](https://github.com/davlgd/biscuit-sandbox/blob/main/CHANGELOG.md), under *Unreleased* until a release.

Write plain, direct English, in the active voice. Keep a sentence under 25 words when possible, and a paragraph to about 80 words. Spell out an acronym at its first use, use the serial comma, and give every code block a language.

Prefer prose to lists, and keep each list item to one line. Write each paragraph on a single line of the source file, and let the editor wrap it: a diff then shows the paragraph that changed.

On GitHub, the repository home shows the README and this guide 838 pixels wide, and the file view other pages 1,012 pixels wide. A list item shouldn't wrap, and the last line of a paragraph shouldn't hold only a few words.

crates.io shows the README files too, where relative links break: use absolute GitHub URLs.

## Check a change

Run these commands before you consider a change done. Clippy must report no warning, and every test must pass:

```bash
cargo fmt
cargo clippy --all-targets -- -D warnings
cargo test
```

A change to the service, or to the container sessions of the bastion, must also pass [`tests/e2e.sh`](https://github.com/davlgd/biscuit-sandbox/blob/main/tests/e2e.sh). Its header explains how to run it. Never commit the addresses, host names, or paths of a deployment.

A change to the crates of biscuit-ssh can break this service: run these checks too after one.

Read your Markdown changes as GitHub renders them, at both widths of [Code rules](https://github.com/davlgd/biscuit-sandbox/blob/main/CONTRIBUTING.md#code-rules), and test the links.

## Commit messages

Write the subject on one line, as a capitalized imperative sentence, without a prefix or a body:

```text
Add the sandbox service: disposable containers reachable through the bastion
Harden sandboxes: reduced capabilities, confirmed startup check, strict renew body, bounded limits
```

## Release

A release commit, `Release 0.1.0`, sets the version in `Cargo.toml`, updates `Cargo.lock`, and moves the *Unreleased* entries of the changelog under the version and its date. Tag it `v0.1.0`.

Publish the release on crates.io after the biscuit-ssh crates it depends on, `biscuit-ssh-core` and `biscuit-ssh-service`, in the versions `Cargo.toml` requires. Until then, check the package against the local checkout:

```bash
cargo package --config 'patch.crates-io.biscuit-ssh-core.path="../biscuit-ssh/crates/core"' \
  --config 'patch.crates-io.biscuit-ssh-service.path="../biscuit-ssh/crates/service"'
```
