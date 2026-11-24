# Free Local LLM on GitHub Actions

This experiment runs an open-weight language model **locally on a GitHub-hosted Ubuntu CPU runner**. It does not call OpenAI, Gemini, Claude, or any hosted inference API, and it requires no API key.

## What it uses

- Runner: `ubuntu-24.04` (GitHub-hosted standard runner)
- Runtime: [Ollama](https://ollama.com/)
- Model: `qwen2:0.5b` (small Qwen 2 model)
- Reusable setup action: [pydantic/ollama-action](https://github.com/pydantic/ollama-action)
- Trigger: manual via `workflow_dispatch`
- Output: printed in the Actions log and uploaded as a downloadable JSON artifact

## Run it

1. Open the [Free Local LLM Inference workflow](../../actions/workflows/free-local-llm.yml).
2. Select **Run workflow**.
3. Enter a prompt (or keep the default VLSI-related prompt).
4. Click **Run workflow** and wait for the job to finish.
5. Open the completed run to read the model output in the logs.
6. Under **Artifacts**, download `local-llm-inference-result` for the structured JSON response and timing metrics.

## Cost and hardware notes

- On a public repository, standard GitHub-hosted runner usage is free under GitHub's public-repository Actions policy.
- The runner uses CPU inference; this workflow does not request or receive a dedicated GPU.
- The model weights are downloaded on the first run. The Ollama action caches its installation and model between eligible runs.
- GitHub-hosted jobs start on fresh runner instances. Cache availability and cache eviction are controlled by GitHub.
- This is a small-model demonstration, not a high-performance assistant. Responses may be limited and CPU inference may take time.
- This workflow has a 30-minute job timeout and a 7-day artifact retention period.

## Security

The workflow has read-only repository permissions. The prompt is supplied as a workflow input and passed through an environment variable rather than interpolated into shell source. Do not put secrets or private data in prompts: workflow inputs and logs may be visible to people with repository access.

## Files

- Workflow: `.github/workflows/free-local-llm.yml`
- This guide: `ai-model-experiments/README.md`
