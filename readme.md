# ollama - shared estate LLM service

The one Ollama instance every salem app shares, deployed as a dedicated Anchor
app (same pattern as [hermes](https://gitea.salem.internal/vohzd/hermes), the
shared mail gateway). Apps never run their own model server; they consume this
one over the anchor network.

## Consuming (any app)

Add to the app's `[apps.env]` (non-secret LAN URL):

```toml
OLLAMA_BASE_URL = "http://ollama:11434"
```

`ollama` is the app's stable network alias on the shared anchor network - it
survives redeploys. Memorymark is the first consumer (summaries, auto-tag,
embeddings); fenmark/domesday/wardstone can point at the same URL when they
grow LLM features. Apps that expose the URL as an admin runtime setting
(memorymark's `/admin/setup`) need no redeploy at all.

## Models

Pulled once into the external `anchor-ollama-models` volume; they survive
redeploys. Current set (CPU-sized for the 5500U - a 120-word summary lands in
roughly 45-90 s):

| model | size | used for |
|---|---|---|
| `qwen2.5:3b-instruct` | 1.9 GB | generation (summaries, tagging) |
| `nomic-embed-text` | 274 MB | embeddings |

Pull a model:

```sh
podman exec $(podman ps --format '{{.Names}}' | grep '^anchor-ollama-.*-app$') \
  ollama pull <model>
```

Before moving any consumer to a 7B model, raise its client + job timeouts
(memorymark: `OLLAMA_TIMEOUT_MS` 240s and the 300s summarise job timeout) and
bump `[apps.resources]` memory here to 8G.

## Deploying

From the checkout on salem (`~/apps/ollama`):

```sh
cd ~/apps/ollama && git pull
bun ~/anchor/packages/cli/src/index.ts deploy ollama \
  --config anchor.salem.toml --context ~/apps/ollama
```

The image is a one-line wrapper over `docker.io/ollama/ollama:latest` (Anchor
apps are built, not pulled), so a deploy is effectively a re-tag plus a
container recreate. Rollout is deliberately `recreate`: blue-green would run
two Ollamas side by side and double peak RAM on the 16 GB box.

Resource posture (see `anchor.salem.toml`): one generation at a time, one
model resident, 10-minute keep-alive, 6G cap - this box shares 16 GB with
~20 containers.

## History

Extracted from memorymark's manifest on 2026-07-03 (it started life as
memorymark roadmap item 0.1) so shared infrastructure is not owned by one
consumer's repo.
