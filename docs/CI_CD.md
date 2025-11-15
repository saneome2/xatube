# CI/CD with GitHub Actions

This document explains how the GitHub Actions workflows provided in this repo work, and what secrets/steps are required to use CI and CD (deploy).

## Workflows

- `.github/workflows/ci.yml` - runs on push / PR to `main` and does the following:
  - Runs backend tests (if present)
  - Builds frontend (`frontend`) and runs tests
  - Builds Docker images for backend, frontend and nginx
  - Pushes images to GitHub Container Registry (GHCR)

- `.github/workflows/cd.yml` - runs on push to `main` and does the following:
  - Connects to the remote host via SSH and updates running docker services on the Docker Swarm with the newly built images (uses `docker service update`)

## Required Secrets

Add the following secrets in your repository settings (Settings -> Secrets and variables -> Actions):

- `GHCR_PAT` - Personal Access Token with `write:packages` and `read:packages` permissions to allow publishing images to GHCR. Alternatively, you can use the `GITHUB_TOKEN` if GHCR is configured to accept it.
- `GHCR_USERNAME` - Username for the GHCR login (usually your GitHub username or the registry owner)
- `SSH_PRIVATE_KEY` - The private key for an SSH user that has permission to manage Docker on the remote host.
- `DEPLOY_HOST` - Remote server hostname or IP
- `DEPLOY_USER` - Username on the remote host
- `SSH_PORT` - (optional) port for SSH, defaults to 22

## Server setup (remote host)

1. Create a user with sudo permissions or a user that can manage docker services.
2. Make sure Docker and Docker Swarm are installed and configured (`docker swarm init` performed on manager).
3. Make sure the server ACL will allow `docker service update` (the SSH user should have rights to the docker socket or use `sudo docker ...`).
4. Optional: configure the server to log in to GHCR once, using `docker login ghcr.io` with your GHCR creds, otherwise the CD job will do `docker login` during deploy.

## How it deploys

- The CI workflow publishes images to GHCR with `latest` and `${{ github.sha }}` tags.
- The CD workflow logs into the remote host via SSH, pulls the `latest` images from GHCR, and runs:

  - `docker service update --image ghcr.io/<owner>/xatube-backend:latest xatube_backend`
  - `docker service update --image ghcr.io/<owner>/xatube-frontend:latest xatube_frontend`
  - `docker service update --image ghcr.io/<owner>/xatube-nginx:latest xatube_nginx`

> NOTE: This assumes service names on the Swarm are `xatube_backend`, `xatube_frontend` and `xatube_nginx` respectively—these names match those created by `docker stack deploy -c docker-compose-swarm.yml xatube`.

## Optional: Using docker stack deploy

If you'd prefer to use `docker stack deploy`, either keep the `docker-compose-swarm.yml` updated to reference the `ghcr.io/...` image names or run the following sequence

```bash
# on remote host
docker login ghcr.io -u <GHCR_USERNAME> -p <GHCR_PAT>
# edit docker-compose-swarm.yml to use ghcr images or override
docker stack deploy -c docker-compose-swarm.yml xatube
```

## Notes & Security

- `GHCR_PAT` must be kept secret. Use GitHub repository secrets.
- Avoid storing secrets in plain text in workflows.
- If you need production HTTPS for nginx, set `secure: true` when cookie is set and configure HTTPS in nginx.

## Troubleshooting

- If `docker service update` reports errors, check `docker service ps` and `docker service logs` on the manager.
- If `docker pull` fails, verify GHCR credentials and network access from the remote host.
- If images are not updated on the server, ensure the `docker service update` commands use the correct service names.

If you want, I can also:
- Add a `workflow_dispatch` manual trigger for the `cd` job.
- Add healthcheck steps for deploy to run smoke tests after deployment.
