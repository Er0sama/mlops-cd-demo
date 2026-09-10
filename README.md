# mlops-cd-demo

Tiny Flask inference API used to practise continuous delivery. The prediction
is deliberately dumb (input times two) — the point of the repo is the pipeline
around it, not the model.

## Endpoints

- `GET /` — service name and status
- `GET /health` — status and model version
- `POST /predict` — takes `{"value": 5}`, returns `{"prediction": 10}`

## Running it locally

```bash
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
pytest
python app.py
```

Or in a container:

```bash
docker build -t mlops-cd-demo:local .
docker run --rm -p 5000:5000 mlops-cd-demo:local
```

## How a release works

Merging to `main` doesn't deploy anything. Pushing a version tag does:

```bash
git tag v1.2.0
git push origin v1.2.0
```

That runs `.github/workflows/cd.yml`, which:

1. runs the tests
2. builds one image and pushes it to `ghcr.io/er0sama/mlops-cd-demo` as both
   `1.2.0` and `latest`
3. deploys that image to staging on port 5000 and curls `/health`
4. waits for manual approval, then deploys **the same image** to production on
   port 5001

Nothing is rebuilt between staging and production. Whatever passed the staging
health check is exactly what goes live.

## Deployment target

The tutorial deploys over SSH to a remote Ubuntu box. This repo uses a
self-hosted GitHub Actions runner on the deployment machine instead, because
the host isn't reachable from GitHub's runners. The deploy jobs run `docker`
directly rather than through `appleboy/ssh-action`. Everything else — the
registry, the environments, the approval gate — is unchanged.

Staging and production are two containers on the same machine, `mlops-api-staging`
on port 5000 and `mlops-api-prod` on port 5001.

## Rolling back

Old images are still in the registry, so going back is just running an older tag:

```bash
docker stop mlops-api-prod && docker rm mlops-api-prod
docker run -d --name mlops-api-prod --restart unless-stopped -p 5001:5000 \
  ghcr.io/er0sama/mlops-cd-demo:1.0.0
```

This is why the version tags matter and `latest` isn't enough — you can't ask
for a specific past build if you never named it.
