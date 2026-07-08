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

      - name: Get Alpine Docker WSL export
        uses: actions/checkout@v4
        with:
          repository: BrammyS/github-action-docker-wsl2
          lfs: false
          sparse-checkout: wsl
          path: docker-wsl

      - name: Create LFS file list
        shell: pwsh
        working-directory: docker-wsl
        run: git lfs ls-files -l | ForEach-Object { $_.Split(' ')[0] } | Sort-Object | Set-Content .lfs-assets-id

      - name: Restore LFS cache
        uses: actions/cache@v4
        id: lfs-cache
        with:
          path: docker-wsl/.git/lfs
          key: ${{ runner.os }}-lfs-${{ hashFiles('docker-wsl/.lfs-assets-id') }}
          restore-keys: |
            ${{ runner.os }}-lfs-

      - name: Pull LFS files
        shell: pwsh
        working-directory: docker-wsl
        run: git lfs pull

      - name: Import Alpine and start Docker
        shell: pwsh
        run: |
          New-Item -ItemType Directory -Force -Path "$PWD\AlpineRoot" | Out-Null
          wsl --import Alpine "$PWD\AlpineRoot" "${{ github.workspace }}\docker-wsl\wsl\alpine-docker-wsl.tar"
          # Keep WSL alive between steps
          wsl -d Alpine -- sh -c 'nohup sleep infinity > /dev/null 2>&1 &'
          # Start Docker daemon and wait for it to be ready
          wsl -d Alpine -- sh -c 'nohup dockerd > /var/log/dockerd.log 2>&1 &'
          $timeout = 60; $elapsed = 0
          while ($elapsed -lt $timeout) {
            $null = wsl -d Alpine -- docker info 2>&1
            if ($LASTEXITCODE -eq 0) { break }
            Start-Sleep -Seconds 2; $elapsed += 2
          }
          if ($elapsed -ge $timeout) { Write-Error "Docker daemon failed to start"; exit 1 }

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
| WSL distro | Alpine Linux |
| Docker access | `tcp://127.0.0.1:2375` |
| Storage | Git LFS (`wsl/alpine-docker-wsl.tar`) |

## Repository structure

```
.github/workflows/
  build-wsl-docker-image.yml   # Manual: builds Alpine + Docker WSL export
  wsl2.yml                     # Validation: tests the full flow
wsl/
  alpine-docker-wsl.tar        # Pre-built WSL export (Git LFS)
```

## Requirements

- A `windows-2025` (or compatible) GitHub-hosted runner
- The build workflow must have been run at least once to generate `wsl/alpine-docker-wsl.tar`

To trigger the build, go to **Actions → build-wsl-docker-image → Run workflow** and supply the Alpine version (e.g. `3.24.1`).

## Contributing

Issues and pull requests welcome. If you find improvements to the Alpine setup or Docker configuration, please contribute.
