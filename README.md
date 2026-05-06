# SnapCD Self-Hosted - Docker Deployment

This guide deploys a complete self-hosted SnapCD stack using Docker Compose:

- **SQL Server** — application database and Service Bus transport (single instance, single DB)
- **Redis** — distributed cache used by the Server
- **Snap CD Server** — the control plane (Web UI, API, orchestration, license service)
- **SnapCD Runner** — one execution agent that connects to the Server above

For background on the editions, licensing, and limits, see the [self-hosting docs](https://docs.snapcd.io/self-hosting).

## Prerequisites

- Docker and Docker Compose installed
- Outbound network access to:
  - `snapcd.io` (only required if you want to apply an Enterprise license)
- Terraform and/or OpenTofu binaries installed on the host (mounted into the runner container)

## Quickstart

The committed `config/server/appsettings.json` and `config/runner/appsettings.json` are working out-of-the-box dev configurations — no edits required. The Server pre-seeds an organization, an admin user (`admin@preseeded.io` / `Admin#123`), a Service Principal, and a Runner record on first start, all using the well-known ID `10000000-0000-0000-0000-000000000000`. The bundled Runner config matches those pre-seeded credentials, so the whole stack connects together on first boot without any manual setup.

```bash
# Bring up the entire stack
docker compose up -d

# Tail logs
docker compose logs -f
```

Then visit the Dashboard at `http://localhost:8080` and log in with the pre-seeded credentials.

## Configuration

All configuration files live in the `config/` directory, split per service:

```
config/
├── server/
│   ├── appsettings.json                # committed defaults — edit only if you really mean to
│   └── appsettings.Production.json     # OPTIONAL, gitignored — your environment overrides
└── runner/
    ├── appsettings.json                # committed defaults
    ├── appsettings.Production.json     # OPTIONAL, gitignored — your real Runner ID + secret
    ├── known_hosts
    ├── id_rsa                          # gitignored
    └── preapproved-hooks/
        └── sample.sh
```

### Override pattern

The Server and Runner both run with `ASPNETCORE_ENVIRONMENT=Production` (default in the bundled `docker-compose.yml`), which means ASP.NET Core layers `appsettings.Production.json` **on top of** `appsettings.json`. To override any setting:

1. Create the override file. For example:

   ```bash
   cat > config/runner/appsettings.Production.json <<'JSON'
   {
     "Runner": {
       "Id": "11111111-1111-1111-1111-111111111111",
       "OrganizationId": "22222222-2222-2222-2222-222222222222",
       "Credentials": {
         "ClientId": "my-runner-sp",
         "ClientSecret": "<paste-here>"
       }
     }
   }
   JSON
   ```

2. Uncomment the matching bind-mount line in `docker-compose.yml` (one is pre-staged for both `snapcd-server` and `snapcd-runner`, commented out).

3. Restart the affected service:

   ```bash
   docker compose up -d --force-recreate snapcd-runner
   ```

You only override the keys you want to change. Everything else falls through to the committed `appsettings.json`. Override files are gitignored, so secrets never end up in version control.

### Server defaults

The committed `config/server/appsettings.json` works out of the box for local evaluation. It uses:

- `myPassw0rd` against the `snapcd-sql` service
- `EmailSender.EmailProvider = "NoOp"` (no emails sent — password resets and invites won't be deliverable)
- Redis at `snapcd-redis:6379` for the distributed cache (provided by the bundled `snapcd-redis` service)
- Dev RSA + AES keys for OpenIddict and the secret store

For real deployments you should override at minimum:

| Setting | Why override |
|---------|-------------|
| `ConnectionString` | Point at a managed/HA SQL Server. Match the host-side `SA_PASSWORD` in `docker-compose.yml`. |
| `Server.Host` | Public URL the Server is reachable at (used for redirect URIs and email links). |
| `OpenIdConnect.TokenSigning.RsaPrivateKey` / `RsaPublicKey` | Generate fresh — see below. |
| `OpenIdConnect.TokenEncryption.SymmetricKey` | Generate fresh — see below. |
| `SecretStore.SqlServer.SymmetricKey` | Generate fresh — see below. |
| `EmailSender.EmailProvider` | Switch to `AmazonSES` (and supply credentials) so password resets and invites are deliverable. |
| `License.LicenseServerBaseUrl` | Leave as `https://snapcd.io` unless you run your own license issuer. |

#### Generate fresh signing keys

```bash
# RSA keypair for OpenIdConnect token signing
openssl genrsa -out token-signing.key 2048
openssl rsa -in token-signing.key -pubout -out token-signing.pub

# AES-256 key for OpenIdConnect token encryption (and the same format works for the secret store)
openssl rand -base64 32
```

Paste the values into `config/server/appsettings.Production.json` under the same paths as in `appsettings.json`. The PEM bodies must use literal `\n` separators inside the JSON string (the committed `appsettings.json` shows the exact shape).

> **Treat these as secrets.** They never need to be rotated under normal operation, but the deployment will lose access to encrypted data if they change.

### Runner defaults

The committed `config/runner/appsettings.json` boots, but with placeholder GUIDs — the runner cannot actually connect until you supply a real Runner ID, Organization ID, Client ID and Client Secret. Bring the Server up first, register the first user against the pre-seeded organization, create a Service Principal and a Runner via the Dashboard, then drop those values into `config/runner/appsettings.Production.json`:

| Setting | Description |
|---------|-------------|
| `Runner.Id` | The unique ID of the Runner (GUID, from the Server's Dashboard) |
| `Runner.Instance` | Friendly name for this Runner instance |
| `Runner.Credentials.ClientId` | Service Principal client ID |
| `Runner.Credentials.ClientSecret` | Service Principal client secret |
| `Runner.OrganizationId` | The organization ID the Runner belongs to |
| `Server.Url` | Defaults to `http://snapcd-server:8080` (the in-network Server). Override only if your Server lives elsewhere. |

### SSH Keys (Optional)

If your Terraform/OpenTofu modules use private Git repositories over SSH:

```bash
cp ~/.ssh/id_rsa ./config/runner/id_rsa
chmod 600 ./config/runner/id_rsa
```

The bundled `config/runner/known_hosts` already contains GitHub and GitLab fingerprints. To add more:

```bash
ssh-keyscan your-git-server.com >> config/runner/known_hosts
```

### Engine (Terraform / OpenTofu) Binaries

The runner container expects `terraform` and/or `tofu` to be available at `/usr/local/bin/`. The compose file bind-mounts those paths from your host. Update the paths in `docker-compose.yml` if your binaries live elsewhere:

```bash
which terraform
which tofu
```

> Snap CD strives to support the latest available version of `tofu`. For `terraform` we design for binaries up to release [1.5.7](https://github.com/hashicorp/terraform/releases/tag/v1.5.7), the final release under the [Mozilla Public License 2.0](https://github.com/hashicorp/terraform/blob/v1.5.7/LICENSE).

### Pre-approved Hooks (Optional)

Drop approved hook scripts into `config/runner/preapproved-hooks/` and enable in `config/runner/appsettings.Production.json`:

```json
{
  "HooksPreapproval": {
    "Enabled": true,
    "PreapprovedHooksDirectory": "/app/preapproved-hooks"
  }
}
```

## Running

```bash
# Bring up the whole stack
docker compose up -d

# Tail logs
docker compose logs -f

# Tear down
docker compose down
```

`depends_on` ordering with health gates ensures SQL Server and Redis are healthy before the Server starts, and the Server is up before the Runner connects, so a single `docker compose up -d` is all that's needed.

### Azure Authentication for the Runner

The runner image (`snapcd-runner-azure`) bundles the Azure CLI. To authenticate:

```bash
docker compose exec snapcd-runner /bin/bash
az login
```

Re-run when the login expires. For other clouds, set credentials as environment variables on the `snapcd-runner` service.

## Limits and Licensing

The Server starts in **Community Edition**: 2 stacks, 2 runners, 20 modules, no SSO, no Turnstile. To raise limits, apply an Enterprise license key from `https://snapcd.io/Subscription` via the Dashboard's `/License` page. See the [self-hosting docs](https://docs.snapcd.io/self-hosting) for details.

## Troubleshooting

```bash
# Check container status
docker compose ps

# Tail server logs only
docker compose logs -f snapcd-server

# Tail runner logs only
docker compose logs -f snapcd-runner

# Get a shell in the server container
docker compose exec snapcd-server /bin/bash

# Reset the database (DESTRUCTIVE)
docker compose down -v
```

## Networking

All four services share the bridge network `snapcd-net`. Inside that network:

- `snapcd-server` reaches the database at `snapcd-sql:1433` and the cache at `snapcd-redis:6379`.
- `snapcd-runner` reaches the server at `snapcd-server:8080`.

Only the Server's port 8080 is published to the host (so you can hit the Dashboard at `http://localhost:8080`); the database is also exposed on the host at `localhost:1433` to make `sqlcmd`/SSMS access easy in dev — remove that mapping for production deployments.

## Docker Images

| Service | Image |
|---------|-------|
| `snapcd-sql` | `mcr.microsoft.com/mssql/server:2022-latest` |
| `snapcd-redis` | `redis:7-alpine` |
| `snapcd-server` | `ghcr.io/schrieksoft/snapcd/snapcd-server:1.0.0` |
| `snapcd-runner` | `ghcr.io/schrieksoft/snapcd/snapcd-runner-azure:1.0.0` |

Pin versions explicitly for production. Available tags are listed at [github.com/schrieksoft/snapcd/pkgs/container/snapcd%2Fsnapcd-server](https://github.com/schrieksoft/snapcd/pkgs/container/snapcd%2Fsnapcd-server) and […/snapcd-runner-azure](https://github.com/schrieksoft/snapcd/pkgs/container/snapcd%2Fsnapcd-runner-azure).
