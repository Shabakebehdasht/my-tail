# 9router backup

This folder holds the 9router state that the `maity` workflow automatically
restores onto each fresh runner, so you never have to re-do the dashboard setup
(provider = opencode, model = hy3, combo = code, copied API key).

## Files committed here

- `data.sqlite`  -> copied to `~/.9router/db/data.sqlite`
- `jwt-secret`   -> copied to `~/.9router/jwt-secret`

Both are backed up straight from the source server (`~/.9router/`).

## How to refresh the backup from the source server

On the server where 9router is already configured the way you like:

```bash
# stop 9router first so the db is not locked
cp ~/.9router/db/data.sqlite  /path/to/my-tail/9router/data.sqlite
cp ~/.9router/jwt-secret       /path/to/my-tail/9router/jwt-secret
cd /path/to/my-tail
git add 9router/data.sqlite 9router/jwt-secret
git commit -m "Refresh 9router backup"
git push origin main
```

Next workflow run will pick up the new state automatically.

> Note: `data.sqlite` and `jwt-secret` are committed to this repo on purpose
> (free models only, no paid keys). Do NOT put real paid provider keys in here.
