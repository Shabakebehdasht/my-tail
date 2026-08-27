# Hermes Agent backup

Restored automatically by the `maity` workflow (step 13).

## Committed here

- `config.yaml` — the Hermes config. It points the default model at the
  custom OpenAI-compatible provider `http://127.0.0.1:20128/v1` (the 9router
  gateway started earlier in the workflow), model `code`, context `512000`.
  The API key is stored as an **env ref** (`${HERMES_CUSTOM_127_0_0_1_20128_API_KEY}`),
  NOT a literal, so no secret is committed to the repo.

- `seed-message.txt` — the **one-time welcome message** sent to Telegram after
  the gateway starts (a summary of the setup). Edit freely; no secret needed.

## MCP servers & skills (configured by the workflow, not the seed)

The workflow installs and wires these automatically in step 13:

- **CodeGraph** (MCP) — `npm i -g @colbymchenry/codegraph`, then `codegraph init --index`
  in h-dashboard. Registered in `config.yaml` as `command: codegraph, args: [serve, --mcp]`.
- **Laravel Boost** (MCP) — `composer require laravel/boost`, served via
  `php artisan boost:mcp`. Registered in `config.yaml`.
- **shadcn/improve** (agent SKILL, not MCP) — `npx -y skills add shadcn/improve`.
- **PostGIS** (MCP) — `uvx mcp-postgis`, connected to the h-dashboard Postgres
  (creds read from h-dashboard/.env at runtime; falls back to the static
  default). Registered in `config.yaml` as `MCP_POSTGIS_MODE: read_only`.

Because these are written by the workflow (deterministically), later Telegram
chats pick them up on the first message — no manual setup needed.

## Secrets required in GitHub repo settings

The workflow reads these from `${{ secrets.* }}` and writes them into
`~/.hermes/.env` at runtime (never committed):

| Secret name                    | Value                                                  |
|--------------------------------|--------------------------------------------------------|
| `HERMES_CUSTOM_API_KEY`        | the 9router API key (the one copied from its dashboard) |
| `TELEGRAM_BOT_TOKEN`           | Telegram bot token from @BotFather                    |
| `TELEGRAM_ALLOWED_USERS`       | Telegram user id — also used as the home channel      |

> `TELEGRAM_HOME_CHANNEL` is intentionally NOT a separate secret: the home
> channel is the same as the allowed user id, so the workflow reuses
> `TELEGRAM_ALLOWED_USERS` for it.

To refresh the config from this server:

```bash
cp ~/.hermes/config.yaml /path/to/my-tail/hermes/config.yaml
cd /path/to/my-tail
git add hermes/config.yaml
git commit -m "Refresh hermes config"
git push origin main
```

> The actual secret values live only in GitHub Secrets (and in the live
> server's ~/.hermes/.env). Do NOT commit a real .env here.
