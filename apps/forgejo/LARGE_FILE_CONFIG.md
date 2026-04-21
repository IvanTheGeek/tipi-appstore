# Forgejo Large File Configuration

## The GitHub Problem

GitHub enforces a hard 100MB per-file limit on all git pushes. Files exceeding this limit
are rejected at the receive-pack level — no bypass exists except Git LFS.

Git LFS was attempted but introduced two new problems:

1. `git lfs migrate import --everything` rewrites all branch history (not just target files),
   causing live repo divergence with force-pushed history
2. LFS objects are uploaded over HTTPS; files 200MB+ regularly hit upload timeouts before
   the transfer completes (connection closed mid-upload, exit 141/SIGPIPE)

The workaround was byte-size splitting (`split -b 45m`) to keep all chunks under 100MB.
Codex session files embed full file diffs per event line, producing files up to 450MB that
compress only ~2:1 (not the typical 10:1), making compression useless as an alternative.

## Why Forgejo Solves This

Forgejo has no built-in per-file size limit on git pushes. Files of any size can be pushed
as long as the HTTP connection stays open. No Git LFS required; no splitting required.

## Required Configuration

Applied via environment variables in `docker-compose.json` (RunTIPI env var format:
`FORGEJO__<SECTION>__<KEY>`).

| app.ini key                  | Env var                              | Value  | Reason                                            |
|------------------------------|--------------------------------------|--------|---------------------------------------------------|
| `[server] READ_TIMEOUT`      | `FORGEJO__server__READ_TIMEOUT`      | `0`    | No HTTP read timeout — large pushes take minutes  |
| `[server] WRITE_TIMEOUT`     | `FORGEJO__server__WRITE_TIMEOUT`     | `0`    | No HTTP write timeout                             |
| `[server] LFS_MAX_BLOB_SIZE` | `FORGEJO__server__LFS_MAX_BLOB_SIZE` | `0`    | No LFS object size limit (0 = unlimited)          |
| `[attachment] MAX_SIZE`      | `FORGEJO__attachment__MAX_SIZE`      | `2048` | Web UI file upload limit in MB                    |

`0` = unlimited for timeout and size values.

## Proxy Note

RunTIPI uses Traefik as its reverse proxy. Traefik has no built-in HTTP body size limit,
so large pushes are not blocked at the proxy layer. No Traefik configuration needed.
