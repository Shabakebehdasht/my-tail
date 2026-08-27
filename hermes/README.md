# Hermes Agent backup

Restored automatically by the `maity` workflow (step 13).

## Committed here

- `config.yaml` — the Hermes config. It points the default model at the
  custom OpenAI-compatible provider `http://127.0.0.1:20128/v1` (the 9router
  gateway started earlier in the workflow), model `code`, context `512000`.
  The API key is stored as an **env ref** (`${HERMES_CUSTOM_127_0_0_1_20128_API_KEY}`),
  NOT a literal, so no secret is committed to the repo.

- `seed-message.txt` — the **one-time setup instructions** executed by Hermes
  right after the gateway starts (in a dedicated `seed-setup` session). It tells
  the agent to cd into h-dashboard, sync branches, read agents.md, configure the
  Laravel boost / codegraph / shadcn-improve MCPs, and set the default PR target.
  Edit the file in the repo to change the instructions; no secret needed.

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
