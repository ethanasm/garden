# garden

My public digital garden — a [Quartz v4](https://github.com/jackyzha0/quartz)
site deployed to GitHub Pages on every push to `main`
(`.github/workflows/deploy.yml`).

## ⚠️ This repo is generated

The source of truth is a **private vault**. Everything under `content/` is
one-way synced from that vault's `public/` folder by a script in my workspace
repo (`scripts/sync-garden.sh`, rsync `--delete`) — any direct edit to
`content/` will be destroyed by the next sync.

**Please don't open PRs against content.** If you spot a typo or want to
respond to a note, open an issue instead.

Framework files (`quartz/`, `quartz.config.ts`, `quartz.layout.ts`) come from
Quartz v4 and are edited here in the normal way — only `content/` is machine-
owned.

## Local preview

```bash
npm ci
npx quartz build --serve
```
