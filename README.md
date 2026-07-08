# Run Linux Docker Containers on a Windows GitHub Hosted Runner

Run **Linux Docker containers** from a **Windows GitHub Actions runner** using WSL2 and Alpine Linux.

This project provides a pre-built Alpine WSL2 image with Docker installed, so your Windows pipeline can use Linux containers without slow installs or Ubuntu compatibility issues.

## How it works

1. **Weekly build** ([`build-wsl-docker-image.yml`](./.github/workflows/build-wsl-docker-image.yml)) creates a ready-to-use Alpine WSL2 export with Docker pre-installed and uploads it as an artifact.
2. **Your pipeline** downloads the artifact, imports it into WSL2, starts Docker, and exposes the daemon to Windows on `tcp://127.0.0.1:2375`.

Windows gets full access to the Linux Docker daemon — you can use `docker build`, `docker run`, `docker compose`, etc. directly from PowerShell.

## Usage in your own pipeline

Add the following steps to your Windows workflow:

```yaml
jobs:
  build:
    runs-on: windows-2025
    env:
      DOCKER_HOST: tcp://127.0.0.1:2375

    steps:
      - uses: actions/checkout@v4

      - name: Enable WSL
        shell: pwsh
        run: |
          wsl --install --no-distribution
          wsl --set-default-version 2

      - name: Get latest Alpine Docker image run ID
        id: get-run-id
        shell: pwsh
        env:
          GH_TOKEN: ${{ github.token }}
        run: |
          $runId = gh run list `
            --repo BrammyS/github-action-docker-wsl2 `
            --workflow build-wsl-docker-image.yml `
            --status success `
            --limit 1 `
            --json databaseId `
            --jq '.[0].databaseId'
          Write-Output "run-id=$runId" >> $env:GITHUB_OUTPUT

      - name: Download Alpine Docker WSL artifact
        uses: actions/download-artifact@v4
        with:
          name: alpine-docker-wsl
          path: ${{ github.workspace }}\wsl-artifact
          run-id: ${{ steps.get-run-id.outputs.run-id }}
          github-token: ${{ github.token }}
          repository: BrammyS/github-action-docker-wsl2

      - name: Import Alpine and start Docker
        shell: pwsh
        run: |
          New-Item -ItemType Directory -Force -Path "$PWD\AlpineRoot" | Out-Null
          wsl --import Alpine "$PWD\AlpineRoot" "${{ github.workspace }}\wsl-artifact\alpine-docker-wsl.tar"
          # Keep WSL alive between steps
          wsl -d Alpine -- sh -c 'nohup sleep infinity > /dev/null 2>&1 &'
          # Start Docker daemon
          wsl -d Alpine -- sh -c 'nohup dockerd > /var/log/dockerd.log 2>&1 & sleep 10 && docker info'

      - name: Install Docker CLI on Windows
        shell: pwsh
        run: choco install docker-cli -y

      # Docker is now available from Windows!
      - name: Use Docker
        shell: pwsh
        run: docker run hello-world:linux
```

## Why Alpine?

Alpine is lightweight, starts instantly, and works reliably in WSL2 on GitHub Actions runners.

## Key details

| Setting | Value |
|---------|-------|
| Runner | `windows-2025` |
| WSL distro | Alpine Linux 3.24 |
| Docker access | `tcp://127.0.0.1:2375` |
| Build schedule | Weekly (Sunday midnight UTC) |
| Artifact retention | 90 days |

## Repository structure

```
.github/workflows/
  build-wsl-docker-image.yml   # Weekly: builds Alpine + Docker WSL export
  wsl2.yml                     # Validation: tests the full flow
```

## Requirements

- A `windows-2025` (or compatible) GitHub-hosted runner
- The weekly build must have run at least once before using the artifact

To trigger the first build manually, go to **Actions → build-wsl-docker-image → Run workflow**.

## Contributing

Issues and pull requests welcome. If you find improvements to the Alpine setup or Docker configuration, please contribute.
