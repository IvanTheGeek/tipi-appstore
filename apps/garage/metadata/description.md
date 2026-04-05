# Garage

Garage is a lightweight, S3-compatible distributed object storage server built for self-hosting. It is AGPL-licensed, low on resources, and designed to run on commodity hardware — single node or clustered.

## Ports

| Port | Purpose |
|---|---|
| 3900 | S3 API (primary — exposed via domain) |
| 3903 | Admin API (local access only) |

## Required setup before first start

Garage requires a config file at `${APP_DATA_DIR}/config/garage.toml` on the host. Create it before installing:

```bash
mkdir -p <runtipi-dir>/app-data/<store-slug>/garage/config

cat > <runtipi-dir>/app-data/<store-slug>/garage/config/garage.toml << 'EOF'
metadata_dir = "/var/lib/garage/meta"
data_dir     = "/var/lib/garage/data"

replication_factor = 1

rpc_bind_addr   = "[::]:3901"
rpc_public_addr = "127.0.0.1:3901"
rpc_secret      = "REPLACE_WITH_64_CHAR_HEX"

[s3_api]
s3_region    = "us-east-1"
api_bind_addr = "[::]:3900"

[admin]
api_bind_addr = "[::]:3903"
EOF
```

Generate the RPC secret:
```bash
openssl rand -hex 32
```

## Post-install: apply layout (single node)

After Garage starts, run once to activate the node:

```bash
# Get the node ID
docker exec garage garage node id

# Apply layout (substitute actual node ID)
docker exec garage garage layout assign <node-id> --zone dc1 --capacity 100G
docker exec garage garage layout apply --version 1
```

## Bucket setup

```bash
# Create buckets
docker exec garage garage bucket create public
docker exec garage garage bucket create private

# Make public bucket publicly readable
docker exec garage garage bucket allow public --read --deny-web

# Create access keys
docker exec garage garage key create ivan-key
docker exec garage garage bucket allow public --read --write --key ivan-key
docker exec garage garage bucket allow private --read --write --key ivan-key
```

## Links

- [Documentation](https://garagehq.deuxfleurs.fr/documentation/)
- [Source](https://git.deuxfleurs.fr/Deuxfleurs/garage)
