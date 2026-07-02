# Run Linux Docker Containers on a Windows GitHub Hosted Runner

An example repository that shows how to use **WSL2 on a Windows GitHub Actions runner** to run **Linux Docker containers**.

This is useful when you need a Windows-based workflow but still want to build, test, or run Linux containers as part of the same job.

## What this repo demonstrates

- Running on a **Windows GitHub-hosted runner**
- Using **WSL2** as the Linux environment
- Starting and using **Docker inside WSL2**
- Running **Linux containers** from a GitHub Actions workflow

## Example workflow

The main example workflow is:

- [`.github/workflows/wsl2.yml`](./.github/workflows/wsl2.yml)

It shows the steps needed to prepare WSL2 and execute Docker commands against a Linux environment from a Windows runner.

## Why this approach

GitHub-hosted Windows runners are great when your workflow needs Windows-specific tooling, but Docker container workloads are often Linux-based. Using WSL2 can bridge that gap and let you:

- Keep a Windows runner where required
- Run Linux-first tooling and container workflows
- Reuse existing Docker-based scripts with minimal changes

## Typical use cases

- Building Linux container images from a Windows-oriented pipeline
- Testing software inside Linux containers while keeping Windows-specific setup steps
- Hybrid CI scenarios where both Windows and Linux environments are needed in one workflow

## Repository structure

```text
.github/
  workflows/
    wsl2.yml
README.md
```

## Getting started

1. Open the example workflow.
2. Review the WSL2 setup steps.
3. Copy the relevant parts into your own workflow.
4. Adjust the Docker commands for your image or test process.

## Notes

- This repository is intended as a **working example**.
- Behavior on GitHub-hosted runners can evolve over time, so test the workflow in your own environment.
- The same general technique may also be useful on compatible self-hosted or Azure-based Windows runners.

## Contributing

If you find a cleaner setup or additional compatibility notes, feel free to open an issue or pull request.
