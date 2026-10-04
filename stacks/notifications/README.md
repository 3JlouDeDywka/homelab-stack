# Notifications Stack

Self-hosted notification stack for the homelab.

## Services

| Service | Image | Purpose |
|---|---|---|
| ntfy | `binwiederhier/ntfy:v2.11.0` | Primary push notification server |
| Gotify | `gotify/server:2.5.0` | Backup push notification service |
| Apprise | `caronc/apprise:v1.5.4` | Multi-channel notification gateway |

## Setup

Set these variables in your local `.env`:

```env
DOMAIN=example.com
GOTIFY_PASSWORD=change-me
NTFY_TOKEN=tk_xxxxxxxxxxxxxxxxx
```

Do not commit real passwords or tokens.

Create an ntfy administrator and access token:

```bash
docker exec -it ntfy ntfy user add --role=admin admin
docker exec -it ntfy ntfy token add admin
```

Start the notification stack:

```bash
docker compose -f stacks/notifications/docker-compose.yml up -d
```

Services are available through Traefik at:

- `https://ntfy.${DOMAIN}`
- `https://gotify.${DOMAIN}`
- `https://apprise.${DOMAIN}`

Anonymous ntfy access is disabled by `config/ntfy/server.yml`.

## Unified notification script

Other homelab scripts should use:

```bash
scripts/notify.sh <topic> <title> <message> [priority]
```

Example:

```bash
export DOMAIN=example.com
export NTFY_TOKEN=tk_xxxxxxxxxxxxxxxxx
scripts/notify.sh homelab-test "Test" "Hello World"
```

`NTFY_URL` can override the default `https://ntfy.${DOMAIN}` endpoint.

## Alertmanager

Alertmanager and ntfy share the Docker `proxy` network.

`config/alertmanager/alertmanager.yml` sends firing and resolved alerts to the
`homelab-alerts` ntfy topic.

Authentication uses `/run/secrets/ntfy_token`, sourced from the
`NTFY_TOKEN` environment variable, so the token is not committed to Git.

## Watchtower

Watchtower uses Shoutrrr for ntfy notifications.

Create a dedicated user:

```bash
docker exec -it ntfy ntfy user add watchtower
docker exec -it ntfy ntfy access watchtower watchtower rw
```

Configure Watchtower:

```env
WATCHTOWER_NOTIFICATIONS=shoutrrr
WATCHTOWER_NOTIFICATION_URL=ntfy://watchtower:PASSWORD@ntfy.${DOMAIN}/watchtower?scheme=https
```

After an update, verify that the `watchtower` topic receives a notification.

## Gitea

Open:

`Settings -> Webhooks -> Add Webhook -> Gitea`

Configure:

- Target URL: `https://ntfy.${DOMAIN}/gitea`
- HTTP method: `POST`
- POST content type: `application/json`
- Authorization Header: `Bearer <NTFY_TOKEN>`
- Select the required repository events
- Enable the webhook

Use **Test Delivery** to verify delivery.

## Home Assistant

Open:

`Settings -> Devices & services -> Add Integration -> ntfy`

Use:

```text
https://ntfy.${DOMAIN}
```

as the service URL.

For protected topics, provide an authorized ntfy username and password and add
the desired topic.

Example:

```yaml
action: notify.send_message
target:
  entity_id: notify.homelab
data:
  message: "Home Assistant test notification"
```

## Uptime Kuma

Open:

`Settings -> Notifications -> Setup Notification -> Ntfy`

Configure:

- Server URL: `https://ntfy.${DOMAIN}`
- Topic: `uptime-kuma`
- Authentication: access token or username/password
- Priority: as desired

Use the built-in **Test** button before attaching it to monitors.

## Gotify

Open:

```text
https://gotify.${DOMAIN}
```

The initial administrator password comes from `GOTIFY_PASSWORD`.

Create a Gotify application in the web UI to obtain an application token for
clients that need Gotify.

## Acceptance checks

- ntfy Web UI is reachable.
- ntfy mobile app receives a test notification.
- `scripts/notify.sh homelab-test "Test" "Hello World"` succeeds.
- Alertmanager firing and resolved alerts reach ntfy.
- Watchtower update notifications reach ntfy.
- Gitea, Home Assistant and Uptime Kuma integration instructions are usable.

## Security

- Anonymous ntfy access is denied.
- Never commit `NTFY_TOKEN`, Gotify passwords or other credentials.
- Use HTTPS for externally exposed endpoints.
- Prefer dedicated users and topics for integrations.
