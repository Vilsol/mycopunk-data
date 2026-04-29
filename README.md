# mycopunk-data

Versioned, gzipped JSON dumps of the Unity game **Mycopunk** — every upgrade,
gear, character, mission, objective, enemy, etc., extracted via the
[MycopunkDumper](https://github.com/Vilsol/MycopunkDumper) BepInEx plugin.

Hosted as a static site on Cloudflare Pages.

## Consuming

Bootstrap from the index, then fetch any version:

```js
const idx = await fetch("https://data.mycopunk.example/index.json").then(r => r.json());
const data = await fetch(`https://data.mycopunk.example/data/${idx.latest}/data.json.gz`).then(r => r.json());
```

`fetch` automatically decompresses thanks to the `Content-Encoding: gzip`
header — you get plain JSON in `.json()`. CORS is open (`*`).

## URLs

| Path | Purpose |
|---|---|
| `/index.json` | Version manifest (the only file you need to bootstrap) |
| `/latest/data.json.gz` | 302 → newest version's dump |
| `/data/<version>/data.json.gz` | Specific version's dump (gzipped) |
| `/data/<version>/CHANGES.md` | Human-readable diff vs the previous version |
| `/schema/data.schema.json` | Current JSON Schema (Draft 2020-12) for `data.json` |

`<version>` is the game's `Application.version` (e.g. `v1.7.3`, `v1.8`).

## `index.json`

```json
{
  "latest": "v1.8",
  "schema": "/schema/data.schema.json",
  "versions": [
    {
      "version": "v1.8",
      "buildId": "22983270",
      "dumpedAt": "2026-04-28T22:22:36Z",
      "dumperCommit": "abc1234",
      "size": 3392360,
      "sha256": "f3c2…"
    }
  ]
}
```

`versions` is sorted newest-first. `size` and `sha256` are of the **gzipped**
file as served. `dumperCommit` is the MycopunkDumper git sha that produced the
dump.

## Caching

| Path glob | `Cache-Control` |
|---|---|
| `/data/*/data.json.gz` | `public, max-age=31536000, immutable` (paths are version-pinned, never mutate) |
| `/schema/*.json` | `public, max-age=300` |
| `/index.json` | `public, max-age=60` |

Consumers should re-fetch `index.json` periodically and only fetch new dumps
when `latest` changes.

## Adding a version

Run from the `MycopunkDumper` checkout (game must be installed and BepInEx
configured — see that repo's CLAUDE.md):

```bash
mise run dump
mise run release-dump /path/to/mycopunk-data
```

That task: dumps the game, gzips the output, generates `CHANGES.md` against
the previous version, refreshes `index.json` and `_redirects`, copies the
current schema, and prints a commit recipe. Push triggers Cloudflare Pages
deploy automatically; the GitHub Actions CI validates every dump on each push.
