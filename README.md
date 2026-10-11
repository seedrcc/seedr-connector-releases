# Connect your NAS to Seedr

Seedr Connector makes selected NAS folders available to your Seedr account.
It opens an outbound encrypted connection; your NAS needs internet access to
Seedr on TCP 443, with DNS working. You do not need port forwarding.

Current release: **1.0.0**.

- Container image (amd64, arm64, armv7): `ghcr.io/seedrcc/seedr-connector:1.0.0`
  (`:latest` follows the newest release; pin `1.0.0` for predictable updates).
- Linux binaries: <https://github.com/seedrcc/seedr-connector-releases/releases/tag/v1.0.0>.
  Verify downloaded binaries with `SHA256SUMS`.

For a later release, replace `1.0.0` below with the version shown on the
[releases page](https://github.com/seedrcc/seedr-connector-releases/releases).

## Prepare your folders

Choose a NAS account with access to the folders you want to share. The connector
runs as that account's numeric UID and GID, not root. On a shell, `id NAS_USER`
shows these numbers. NAS ACLs must also allow that account to access the share.

Create a private connector directory containing `config.json` and a `state`
subdirectory. Give `state` mode 0700 and ownership matching the connector's
UID/GID. It stores your pairing and must survive container recreation and NAS
restarts. Protect its backups like passwords. Do not share one state directory
between two installations.

Save this as `config.json`:

```json
{
  "name": "Home NAS",
  "folders": [
    {"name": "data", "path": "/shares/data", "read_only": false}
  ]
}
```

`name` identifies this NAS. Each folder's `name` is the label Seedr sees; `path`
is its path inside the container. Use a separate bind mount and folder entry for
each share. For a read-only folder, set `read_only` to `true` **and** make its NAS
bind mount read-only. Host paths stay on your NAS.

| NAS file or directory | Container path | Access |
| --- | --- | --- |
| `config.json` | `/config/config.json` | Read-only |
| Private `state` directory | `/state` | Read/write |
| Chosen NAS share | `/shares/data` | Read/write, or read-only |

Use bridge networking, no published ports, and no privileged mode. Mount only
chosen shares, configuration and state. Leave the Docker socket unmounted.
The image has no shell; use container logs and its `pair`/`run` commands.

## Pair with your Seedr account

First run the container with this command (keep its default entrypoint):

```text
pair --config /config/config.json --state /state
```

Read its logs. Open the printed HTTPS verification address in your browser,
sign in to **your** Seedr account, and enter the eight-character code printed by
the connector. The page says **Connect your device**, with **Enter Code** and
**Approve** steps. Choose **Verify Code**, check the account shown, then review
the application's name and folder-access permission. Choose **Allow** to approve,
or **Deny** to cancel. Entering the code alone does not approve access.

Wait for the connector to print `Paired. Run seedr-connector run.` Then change
its command to:

```text
run --config /config/config.json --state /state
```

Start it with automatic restart enabled. Its timestamped log should show `online`.
Keep this running while using the NAS through Seedr. Your NAS exports still need
to be selected as storage in Seedr; pairing alone does not create a storage
mount. If your Seedr storage screen does not offer the paired NAS and export
selection, contact Seedr support: that account cannot complete setup through
the current screen. Do not substitute a public NAS WebDAV address.

## Synology Container Manager

Install Container Manager from Package Center. Create a connector folder, for
example `/volume1/docker/seedr-connector`, and put your configuration and state
there. Use absolute paths such as `/volume1/media` for shares.

In **Registry → Settings → Add**, add the registry URL `https://ghcr.io`
and name it `ghcr.io`. Use the full image reference
`ghcr.io/seedrcc/seedr-connector:1.0.0`. If your DSM release cannot search
that registry, use the Compose project below; its explicit image reference
pulls directly from ghcr.io. [Synology registry settings](https://kb.synology.com/en-us/DSM/help/ContainerManager/docker_registry?version=7).

In **Project → Create**, name the project `seedr-connector`, choose its working
folder, and upload a Compose file or use the editor. Use the Compose example
below with your absolute paths and numeric UID/GID. Leave Web Station/web portal
disabled. For pairing, temporarily set `command` to the `pair` command above and
`restart: "no"`; start the project and read its container logs. After approval,
edit its YAML to restore the `run` command and `restart: unless-stopped`, then
recreate/start it with the same state directory. [Synology project guide](https://kb.synology.com/en-us/DSM/help/ContainerManager/docker_project?version=7).

## QNAP Container Station

Install Container Station. Store configuration and state in a persistent share,
for example `/share/Container/seedr-connector`; find the actual absolute path of
the data share rather than guessing its storage-volume name.

In Container Station 3, open **Applications → Create**, name the application,
and paste the Compose example below with absolute paths and numeric UID/GID.
Choose **Validate**, then **Create**. Pair first using `command: [pair, --config,
/config/config.json, --state, /state]` and `restart: "no"`; read the container
logs and approve in your browser. Use the application's **Recreate** action to
restore the run command and restart policy while preserving its bind mounts.
Leave the default web URL port disabled. [QNAP application guide](https://www.qnap.com/en-us/how-to/tutorial/article/how-to-use-container-station-3).

## TrueNAS SCALE

Use **Apps → Discover Apps → Custom App** on Docker-based SCALE releases.
Set image repository `ghcr.io/seedrcc/seedr-connector` and tag `1.0.0`. Configure a non-root user/group
matching your dataset permissions, then add the three host-path mounts from the
table above, using paths under `/mnt/POOL/DATASET`. Keep host networking and
privileged mode disabled, with no ports.

Set the arguments to `pair --config /config/config.json --state /state`, run it,
and approve through its logs. Edit the app to use the run command afterward,
retaining state. Where **Install via YAML** is available, the Compose example
below provides the equivalent setup. Older Kubernetes-based SCALE releases
have different app settings; use the Linux binary if their custom-app options
cannot preserve this non-root setup. [TrueNAS custom-app guide](https://www.truenas.com/docs/scale/apps/installcustomappscreens/).

## Unraid

Enable Docker, then choose **Docker → Add Container**. Name it `seedr-connector`
and enter `ghcr.io/seedrcc/seedr-connector:1.0.0` as Repository. Use bridge networking, privileged
mode off, no ports, and the three path mappings above. A persistent directory
such as `/mnt/user/appdata/seedr-connector` can hold configuration and state;
map only the selected share, for example `/mnt/user/media`, to `/shares/data`.

In Advanced View, set Extra Parameters to `--user UID:GID --read-only
--cap-drop ALL --security-opt no-new-privileges:true` with your actual nonzero
UID/GID. Set Post Arguments to the pair command, apply and read logs to approve.
Then edit Post Arguments to the run command and enable autostart. The connector
does not read `PUID`/`PGID` variables; use `--user` to set its identity.
[Unraid container settings](https://docs.unraid.net/unraid-os/using-unraid-to/run-docker-containers/managing-and-customizing-containers/).

## Plain Docker or Docker Compose

Use absolute paths and replace `1000:1000` with your NAS UID/GID. For a read-only
export add `:ro` to the share mount as well as `read_only: true` in JSON.

```yaml
services:
  connector:
    image: "ghcr.io/seedrcc/seedr-connector:1.0.0"
    user: "1000:1000"
    command: [run, --config, /config/config.json, --state, /state]
    restart: unless-stopped
    read_only: true
    cap_drop: [ALL]
    security_opt: [no-new-privileges:true]
    volumes:
      - /absolute/connector/config.json:/config/config.json:ro
      - /absolute/connector/state:/state
      - /absolute/nas/share:/shares/data
```

Save as `compose.yaml`. Pair once, approve in your browser, then start:

```sh
docker compose run --rm connector pair --config /config/config.json --state /state
docker compose up -d
docker compose logs --tail 20 connector
```

The repository's `compose.yaml` instead reads `SEEDR_CONNECTOR_IMAGE`, `NAS_UID`,
`NAS_GID` and `NAS_SHARE_PATH`. Set the image variable to `ghcr.io/seedrcc/seedr-connector:1.0.0`.
An OCI archive supplied offline can be loaded with a compatible Docker engine
or `podman load -i seedr-connector.oci.tar`; use the reference reported by load.

Without Compose, pair using `docker run --rm --user UID:GID --read-only
--cap-drop ALL --security-opt no-new-privileges:true` with the same three `-v`
mounts and `ghcr.io/seedrcc/seedr-connector:1.0.0 pair --config /config/config.json --state /state`. For the
normal run, add `-d --name seedr-connector --restart unless-stopped` and replace
`pair` with `run`. Do not enable restart for the one-time pairing container.

## Plain Linux binary

Choose the artifact matching `uname -m`:

| CPU | Artifact |
| --- | --- |
| `x86_64` | `seedr-connector-linux-amd64` |
| `aarch64` | `seedr-connector-linux-arm64` |
| `armv7l` | `seedr-connector-linux-armv7` |

Download that binary, `SHA256SUMS`, `LICENSE`, `THIRD_PARTY_NOTICES` and the
`licenses/` directory from `https://github.com/seedrcc/seedr-connector-releases/releases/tag/v1.0.0`. From their
folder, verify the downloaded artifact before executing it:

```sh
sha256sum --ignore-missing -c SHA256SUMS
chmod 755 seedr-connector-linux-amd64
./seedr-connector-linux-amd64 version
```

Require an `OK` result for your binary. Run it as the NAS share's unprivileged
account. Set folder paths in `config.json` to real host paths, not container
paths, and keep state private:

```sh
mkdir -m 700 state
./seedr-connector-linux-amd64 pair --config ./config.json --state ./state
./seedr-connector-linux-amd64 run --config ./config.json --state ./state
```

Use the NAS service supervisor for the run command, with the same user,
configuration and persistent state path. Set `SSL_CERT_FILE` to the NAS's trusted
CA bundle if Linux cannot locate it. Keep certificate verification enabled.

## Reconnect, update and disconnect

Temporary network failures reconnect automatically with randomized backoff.
After a NAS restart, run with the same state directory; do not pair again.
For an update, stop the connector, retain configuration/state, replace the image
or verified binary, and start it again with the same UID/GID. With Compose,
change the image tag, then run `docker compose pull && docker compose up -d`.

Disconnecting/revoking in Seedr stops access to that connector and its active
transfers. Stop its container/service too. To reconnect after revocation, run
pair and approve again using the existing state directory. Two NAS devices can
use one Seedr account, but each needs its own state directory and name.

If pairing reports `invalid_client`, verify the supplied client ID with Seedr
support. `cannot open configuration` means the config path is missing or
unreadable. Credential/state errors mean ownership or 0700/0600 permissions
need correction. `revoked` requires re-pairing. A message that your plan's connector limit is
reached, or that your plan does not include connectors, means the connector is
waiting: it re-checks once an hour, so upgrade or remove another connector and
it connects without a restart. If the log never
shows online, check outbound DNS, HTTPS/WebSocket connectivity, system time and
CA certificates. The connector uses production Seedr endpoints by default.

Folder changes require a restart; the connector updates the export list
without another approval. A `revoked` message requires pairing again. An
`upgrade_required` message names the minimum version to install.
