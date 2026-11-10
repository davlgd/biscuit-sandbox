# AGENTS.md

This file points coding agents to the instructions of the sandbox service, whose rules all apply to agents. Before you change anything, read [CONTRIBUTING.md](https://github.com/davlgd/biscuit-sandbox/blob/main/CONTRIBUTING.md): the architecture, the invariants of the state and the reaper, and how to check a change.

## Before you report a change as done

- Run the [checks](https://github.com/davlgd/biscuit-sandbox/blob/main/CONTRIBUTING.md#check-a-change): format, Clippy, and tests
- Run `tests/e2e.sh` when the change affects the service or the container sessions of the bastion
- Update the doc comments, the README, and the changelog in the same change
- Write the commit subject with the [commit conventions](https://github.com/davlgd/biscuit-sandbox/blob/main/CONTRIBUTING.md#commit-messages)

Report a check you couldn't run, such as the end-to-end tests without a test deployment, rather than skipping it silently.

The service depends on the crates of biscuit-ssh, in `../biscuit-ssh`: change them there, with the rules of that repository.
