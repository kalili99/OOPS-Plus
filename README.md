# OOPS+

## Description

OOPS+ is an extension of the original OOPS! ontology pitfall scanner. The original OOPS! application provides the immediate ontology evaluation report, while OOPS+ adds a second report pipeline for checks that require an LLM.

The current setup contains two web applications:

- **OOPs**: the regular OOPS! application. It generates the normal report immediately.
- **OOPsPlus**: the extended application. It processes OOPS+ jobs and generates the LLM-based report.

When the user enables OOPS+ in the OOPs web interface, OOPs stores the submitted ontology in a shared analysis directory, creates a job, and publishes it to RabbitMQ. An OOPS+ worker consumes that job, runs the OOPsPlus analysis, writes the generated HTML report, and updates the job status. The OOPS+ report page reads that status and displays the report when it is ready.

## Architecture

The root Docker Compose file starts:

- `web-app-oops`: normal OOPS! web app, available on `http://localhost:8080`.
- `web-app-oops-plus`: OOPS+ report viewer, available on `http://localhost:8081`.
- `oops-plus-worker`: background worker that consumes ontology jobs from RabbitMQ.
- `rabbitmq`: job queue used to communicate between OOPs and OOPsPlus.
- `ollama`: local LLM runtime used by the OOPS+ checkers.
- `ollama-pull-model`: helper container that pulls the configured LLM model.

Analysis files are shared through:

```text
./data/oops_analyses
```

This directory contains each OOPS+ analysis folder, including the ontology, status file, and generated report.

## Deployment

Create the environment file from the example:

```bash
cp .env.example .env
```

On Windows PowerShell:

```powershell
Copy-Item .env.example .env
```

Check the values in `.env` :

```env
LLM_IP=http://ollama:11434
LLM_MODEL=gemma3:4b

RABBITMQ_HOST=rabbitmq
RABBITMQ_PORT=5672
RABBITMQ_USER=oops
RABBITMQ_PASSWORD=oops
RABBITMQ_OOPS_PLUS_JOBS_QUEUE=oops.plus.analysis
```

Start the full system:

```bash
docker compose up -d --build
```

The first run may take longer because the Ollama model has to be downloaded.

### Downloading and changing Ollama models

The model used by OOPS+ is configured in `.env`:

```env
LLM_MODEL=gemma3:4b
```

Changing `LLM_MODEL` only tells the OOPS+ worker which model to use. The model must also exist in the shared Ollama volume. If you choose a model that has not been used before, pull it once with the helper service:

```bash
docker compose pull ollama ollama-pull-model
docker compose run --rm ollama-pull-model
```

Then check that the model is available:

```bash
docker compose exec ollama ollama list
```

For example, after pulling both `gemma3:4b` and `gemma4:12b`, the list should contain entries like:

```text
NAME          ID              SIZE
gemma3:4b     ...             3.3 GB
gemma4:12b    ...             7.6 GB
```

If you change `.env` to a model that is already listed by `ollama list`, you do not need to download it again. Recreate the services that read the environment variable so they pick up the new value:

```bash
docker compose up -d --build web-app-oops web-app-oops-plus oops-plus-worker
```

If the OOPS+ worker logs show an error like:

```text
model 'gemma4:12b' not found
```

then the configured model is not available in the Ollama volume. Run the pull helper again and check its output:

```bash
docker compose run --rm ollama-pull-model
```

If Ollama says the model requires a newer version, update the Ollama image first:

```bash
docker compose pull ollama ollama-pull-model
```

and then retry the pull helper.

Useful URLs:

- OOPS!: `http://localhost:8080/oops-0.3.0-SNAPSHOT/`
- OOPS+ report viewer: `http://localhost:8081/oops-0.3.0-SNAPSHOT/`
- RabbitMQ management UI: `http://localhost:15672`

Default RabbitMQ credentials are defined in `.env`.

## Useful Commands

View worker logs:

```bash
docker compose logs -f oops-plus-worker
```

Stop the system:

```bash
docker compose down
```

Rebuild after code changes:

```bash
docker compose up -d --build
```

Optionally run multiple OOPS+ workers to process several ontology jobs in parallel:

```bash
docker compose up -d --scale oops-plus-worker=3
```

Keep in mind that the LLM runtime can become the bottleneck when several workers run at the same time.
