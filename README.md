# biscuit-sandbox

This service creates disposable Linux containers, called sandboxes, that you reach over SSH through the bastion of biscuit-ssh only. You ask its REST API for a sandbox of a Docker Hub image. The service starts it, then answers with a token valid for one hour, from your address only. With this token, you open SSH sessions in the sandbox through the bastion, until the service removes it.

```text
client ── POST /v1/sandboxes {"image": "alpine"} ─▶ sandbox  (pulls, starts, mints a token)
client ── ssh + token ───────────────────────────▶ bastion  (checks the token)
                                                     └──────▶ container (docker exec)
```

The token is a [Biscuit](https://www.biscuitsec.org/), a signed credential the bastion checks. The service needs a biscuit-ssh bastion with container support, and clients connect with the `biscuit-ssh` client. Both come from the biscuit-ssh repository, whose README explains how to install and use them.

## Quick start

You need `curl`, `jq`, the `biscuit-ssh` client, and an SSH key. Any key works: the bastion requires one, but doesn't identify you by it. If the service uses a certificate from a private authority, add `--cacert ca.crt` to `curl`.

```bash
API=https://sandbox.example.com:8444/v1/sandboxes
ssh-keygen -q -t ed25519 -N '' -f sandbox_key

# Create a sandbox for 10 minutes: pulling the image can take minutes
curl -4 -sS "$API" -d '{"image": "python:3.13-slim", "ttl": 600}' > sandbox.json
NAME=$(jq -r .name sandbox.json)
jq -r .token sandbox.json > "$NAME.token"
jq -r .bastion_known_hosts sandbox.json > known_hosts

# Run commands, as many times as needed, or a shell with `sb` alone
sb() {
    biscuit-ssh ssh --bastion "$(jq -r .bastion sandbox.json)" -t "$NAME.token" -i sandbox_key \
        "$(jq -r .login sandbox.json)" -4 -o UserKnownHostsFile=known_hosts -o ConnectTimeout=15 "$@"
}
sb 'python3 --version' </dev/null
echo 'print(6 * 7)' | sb 'python3 -'

# Remove it now rather than at expiration
curl -4 -sS -X DELETE "$API/$NAME"
```

The `known_hosts` file lets `ssh` trust the bastion without a prompt. Without `-i`, the client uses `~/.ssh/id_ed25519`. The `command` field of the response holds a shorter connection command, ready to paste from the directory of the token file, without `-i` or the `known_hosts` file.

The token only accepts the address you requested it from, given in `client_ip`. `curl -4` requests from your IPv4 address, and `-4` keeps the SSH connection on the same address family. Use `-6` for both over IPv6.

## What a sandbox is

### Environment

A sandbox is a Linux container of the image, where sessions run as `root`, with `bash` if the image has it, `sh` otherwise. There is no `sftp`: copy files through a session, such as with `cat app.py | sb 'cat > app.py'`.

A sandbox has no network but loopback, so package managers such as `apk`, `apt`, or `pip` fail with DNS errors. Use an image that already has the tools and packages you need.

### Limits

The service bounds the memory, CPU, and processes of a sandbox, as configured, with no swap. It caps its logs at 1 MB. Commands can't escalate privileges, with `no-new-privileges`, and only have these capabilities:

```text
CHOWN DAC_OVERRIDE FOWNER FSETID KILL SETGID SETUID
```

Commands manage their own files and processes, but can't open raw sockets, use devices, mount, or `chroot`.

### Lifetime

A sandbox lives as long as its token. A token lasts `ttl` seconds from its creation or renewal, from 60 seconds to one hour by default. When it expires, or when you delete the sandbox, the service removes the container with everything in it. The service checks expirations every 10 seconds, and retries while Docker fails.

Renewing gives a new token, valid `ttl` seconds from now, one hour by default. A renewal must extend the life of the sandbox by at least 30 seconds, and renewing before each expiration keeps a sandbox indefinitely.

The service revokes the previous token at once. The bastion then cuts the sessions opened with it at its next check, within 30 seconds by default, and new sessions need the new token. Commands already started keep running: only removing the container stops them.

### Sharing

Any client allowed to use the API can list, renew, and delete every sandbox. Keep the `name` of yours.

## API

Requests only succeed from allowed addresses: any other one gets `401 {"error": "authentication failed"}`, whatever the request. Bodies are JSON of at most 4 KiB, whatever their content type. Responses are JSON, never cacheable, with errors as `{"error": "..."}`.

| Request                                              | Answer                                                                             |
| ---------------------------------------------------- | ---------------------------------------------------------------------------------- |
| `POST /v1/sandboxes` `{"image": "...", "ttl": 3600}` | `201` and the sandbox below, with `ttl` optional, in seconds                       |
| `GET /v1/sandboxes`                                  | `200` and `[{"name", "image", "status", "expires_at", "client_ip"}]`, never tokens |
| `GET /v1/sandboxes/{name}`                           | `200` and one of them, or `404` once it's gone                                     |
| `POST /v1/sandboxes/{name}/renew` `{"ttl": 3600}`    | `200` and a new token for the calling address, as for a creation                   |
| `DELETE /v1/sandboxes/{name}`                        | `204`: the service revokes the token, removes the container, and sessions end      |

The body of a renewal is optional. Only an `Active` sandbox accepts a renewal: any other one gets `404`, and a renewal that extends its life by less than 30 seconds gets `409`.

A creation goes on if its client disconnects, and the sandbox expires as usual. Creations and renewals answer:

```json
{
  "name": "sb-6f1c…",
  "image": "docker.io/library/alpine:3.20",
  "image_id": "sha256:…",
  "expires_at": 1790000000,
  "client_ip": "203.0.113.7",
  "login": "root@sb-6f1c…",
  "bastion": "bastion.example.com:2200",
  "bastion_host_key": "SHA256:…",
  "bastion_known_hosts": "[bastion.example.com]:2200 ssh-ed25519 AAAA…",
  "token": "En0K…",
  "command": "biscuit-ssh ssh --bastion bastion.example.com:2200 -t sb-6f1c….token root@sb-6f1c… -4"
}
```

`expires_at` is the first second the token no longer allows, in Unix time. In listings, `status` is `Creating` while the service pulls and starts the image, with `expires_at` set to `null`. It's `Active` once the sandbox runs, and `Deleting` while the service removes it.

Images come from Docker Hub only, such as `alpine`, `alpine:3.20`, or `grafana/grafana`, with an optional `@sha256:` digest. They must be Linux images that provide `/bin/sh`, whose command keeps running. The service runs containers like `docker run -dit`, so an image whose command is a shell works.

| Error        | Meaning                                                                                                                       |
| ------------ | ----------------------------------------------------------------------------------------------------------------------------- |
| `400`        | Malformed request, `ttl` out of bounds, or invalid image reference: the message says which                                    |
| `404`        | Unknown sandbox, sandbox not active for a renewal, or image not found on Docker Hub                                           |
| `405`, `413` | Wrong method, or body over 4 KiB                                                                                              |
| `409`        | Deletion of a sandbox still in creation, or a renewal extending its life by less than 30 seconds                              |
| `422`        | The image can't be a sandbox: no `/bin/sh`, a command that exits at once, no Linux variant, or a container that doesn't start |
| `429`        | Too many sandboxes, or creations in progress: retry later                                                                     |
| `502`        | Docker Hub failed, or the pull timed out: retry later                                                                         |
| `503`        | Docker failed: retry later                                                                                                    |

## Run the service

### Build and install

The service builds against crates of biscuit-ssh, from crates.io. It needs stable Rust, and runs on Linux:

```bash
cargo build --release
sudo install target/release/biscuit-sandbox /usr/local/bin/
```

The service needs the Docker socket, an Ed25519 signing key of its own, and a state directory. Create the key with the `biscuit-ssh` client: its public key goes to the `[containers]` section of the bastion.

```bash
sudo biscuit-ssh admin keygen -o /etc/biscuit-ssh/sandbox.key
sudo chown biscuit-ssh: /etc/biscuit-ssh/sandbox.key && sudo chmod 600 /etc/biscuit-ssh/sandbox.key
sudo install -d -o biscuit-ssh -g biscuit-ssh /var/lib/biscuit-sandbox
```

### Configure it

Write `/etc/biscuit-ssh/sandbox.toml`. Its `revocation_list` is the one the `[containers]` section of the bastion reads, and `bastion.host_key` the host public key of the bastion:

```toml
listen = "0.0.0.0:8444"
allowed_clients = ["203.0.113.7"]
signing_key = "/etc/biscuit-ssh/sandbox.key"
state_dir = "/var/lib/biscuit-sandbox"
revocation_list = "/var/lib/biscuit-sandbox/revoked"

[bastion]
address = "bastion.example.com:2200"
host_key = "ssh-ed25519 AAAA..."

[tls]
certificate = "/etc/biscuit-ssh/tls.crt"
key = "/etc/biscuit-ssh/tls.key"
```

| Optional setting       | Default                  | Meaning                                                    |
| ---------------------- | ------------------------ | ---------------------------------------------------------- |
| `account`              | `"root"`                 | Account sessions run as, which must exist in the image     |
| `label`                | `"biscuit-ssh.sandbox"`  | Label of the containers, the `label` of the bastion        |
| `limits.cpus`          | `1.0`                    | CPUs of a sandbox, at most 1024                            |
| `limits.disk_mb`       | Unbounded                | Size of the writable layer of a sandbox, in MB             |
| `limits.lifetime`      | `3600`                   | Longest and default `ttl`, from 60 to 3600 seconds         |
| `limits.max_creations` | `2`                      | Creations in progress at once                              |
| `limits.max_sandboxes` | `5`                      | Sandboxes at once, including those in creation or deletion |
| `limits.memory_mb`     | `512`                    | Memory of a sandbox, in MB, without swap                   |
| `limits.pids`          | `256`                    | Processes of a sandbox                                     |
| `limits.pull_timeout`  | `300`                    | Longest image pull, in seconds                             |
| `socket`               | `"/var/run/docker.sock"` | Docker socket                                              |

Write `account`, `label`, and `socket` at the top level, before any section, and the `limits` settings in a `[limits]` section. `allowed_clients` can't be empty, and the service refuses to start with a limit out of bounds.

Keep the `[tls]` section: without it, the service serves plain HTTP, and tokens travel in clear. The service identifies clients by their connection address: behind a reverse proxy, every client would share the proxy's address.

By default, nothing bounds the disk that images and writes inside containers use. Give Docker a file system of its own, with its `data-root`, so a sandbox can't fill the one of the host. Where the storage of Docker supports it, overlay2 on the XFS file system with project quotas, set `limits.disk_mb` to cap the writes of each container. Elsewhere, this setting makes every creation fail with `422`.

### Configure the bastion

Add a `[containers]` section to the configuration of the bastion, with the public key `admin keygen` printed:

```toml
[containers]
root_public_key = "ed25519/..."
revocation_list = "/var/lib/biscuit-sandbox/revoked"
label = "biscuit-ssh.sandbox"
```

The bastion also needs access to Docker, such as with `SupplementaryGroups=docker` in its systemd unit.

### Run it

Check the configuration with `biscuit-sandbox --check --config /etc/biscuit-ssh/sandbox.toml`, as the `biscuit-ssh` account. It loads the signing key, reaches Docker, and loads the TLS certificate, then exits without listening or touching the state. Run one instance per state directory, for example with this systemd unit:

```ini
[Service]
User=biscuit-ssh
SupplementaryGroups=docker
ExecStart=/usr/local/bin/biscuit-sandbox --config /etc/biscuit-ssh/sandbox.toml
Restart=on-failure
NoNewPrivileges=yes
ProtectSystem=strict
ProtectHome=yes
ReadWritePaths=/var/lib/biscuit-sandbox

[Install]
WantedBy=multi-user.target
```

Keep `revocation_list` under `ReadWritePaths`, since the service rewrites it after each change. The service logs to stderr, with the verbosity `RUST_LOG` sets, and never logs tokens.

After a restart, the service finishes what it left: it undoes interrupted creations and completes pending deletions. Every 10 seconds, it also removes its own containers that it doesn't know, and never touches other containers.

Keep the state directory: it holds the identifier the service labels its containers with. A service started with an empty one never removes the containers of the previous one.

## Security

The service identifies clients by address only, and they share every sandbox. An address isn't a person: put the service behind a network you control, where addresses identify machines you trust.

The service signs tokens with a key that only reaches containers, so a leaked sandbox key gives no access to the servers of the bastion. Both still hold Docker access, which a compromise of either turns into root on the host.

The revocation list belongs to the service: revoke a sandbox with `DELETE`, not by editing the file. Commands running in a container stop when the service removes the container, not when the bastion cuts a session.

## Contributing

[CONTRIBUTING.md](https://github.com/davlgd/biscuit-sandbox/blob/main/CONTRIBUTING.md) explains how to build, test, and document the project.

## License

Copyright 2026 davlgd. Licensed under the [Apache License, Version 2.0](https://github.com/davlgd/biscuit-sandbox/blob/main/LICENSE).
