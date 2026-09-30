# Pradnya (NICAI) Containerization and CI/CD

## Architecture

Git push to `main` → validate Compose → build/tag/push backend and frontend images to Docker Hub → SSH to VM → pull images → deploy with Docker Compose → HTTP health checks → release history → rollback attempt if deployment fails.

## Images

- `bhiv/pradnya-backend:<7-character-git-sha>`
- `bhiv/pradnya-frontend:<7-character-git-sha>`

The backend runs FastAPI/Uvicorn on port 8000. The React/Vite frontend is built into static assets and served by Nginx on port 80 inside the container. The VM publishes frontend port 4500 and backend port 8000.

## GitHub Actions secrets

Configure these in **Settings → Secrets and variables → Actions → Secrets**:

- `DOCKER_USERNAME`
- `DOCKER_PASSWORD` (prefer a Docker Hub access token)
- `VM_IP`
- `VM_USERNAME`
- `VM_PASSWORD`
- `VM_PORT`
- `PRADNYA_FRONTEND_ENV_FILE` — multiline frontend build variables, for example:
  ```
  VITE_NICAI_API=http://YOUR_VM_PUBLIC_IP:8000
  VITE_SAMACHAR_API=
  VITE_MITRA_API=
  ```
- `PRADNYA_ALLOWED_ORIGINS` — browser origin(s), comma-separated, for example:
  `http://YOUR_VM_PUBLIC_IP:4500`

Do not add credentials to `.env.example` or commit real `.env` files.

## VM prerequisites

The VM must have Docker Engine and Docker Compose v2 installed. The VM security group/firewall must allow inbound TCP 4500 for the UI and TCP 8000 for the API. If using a reverse proxy or TLS, update the frontend API URL and allowed origins accordingly.

## Verify after deployment

- UI: `http://YOUR_VM_PUBLIC_IP:4500`
- API health: `http://YOUR_VM_PUBLIC_IP:8000/health`
- API docs: `http://YOUR_VM_PUBLIC_IP:8000/docs`
- Signals: `http://YOUR_VM_PUBLIC_IP:8000/signals`
- Patterns: `http://YOUR_VM_PUBLIC_IP:8000/patterns`

VM diagnostics:
```
cd ~/PRADNYA
docker compose -f docker-compose.production.yml ps
docker compose -f docker-compose.production.yml logs --tail=100
curl -fsS http://localhost:8000/health
curl -I http://localhost:4500/
```

## Important notes

- The React frontend reads `VITE_NICAI_API` at build time. Changing it requires rebuilding the frontend image.
- The current workflow follows the prior app's SSH + Docker Hub architecture and uses password-based SSH for compatibility. SSH keys are preferable where your VM setup permits them.
- The rollback job requires at least one prior successful release entry in `/var/tmp/PRADNYA/RELEASE_HISTORY.md`; first deployment cannot roll back to a known-good image.
- Existing application data/logging code writes files under `/app`. This setup persists `/app/logs`; confirm whether bucket/telemetry artifact files also need durable storage before treating them as long-term audit records.
