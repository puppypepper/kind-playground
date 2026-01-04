# Kind Playground

A small, focused repo for learning specific Kubernetes cases locally using
[kind](https://kind.sigs.k8s.io/).

## What This Repo Is

- A set of minimal, reproducible configs and manifests for Kubernetes learning.
- Designed for quick iteration on local clusters, not for production use.
- Each example is self-contained and documents the specific behavior it targets.

## Prerequisites

- Docker (or another container runtime supported by kind)
- kind
- kubectl

## Getting Started

Clone the repo, pick an example, and follow instructions in each README.md

```bash
git clone <repo-url>
cd kind-playground
```

## Notes

- Keep examples minimal and scoped to one concept.
- Prefer small, repeatable steps over large manifests.

## Contributing

PRs for new focused examples are welcome. Keep new examples isolated in their
own directory and include a short README or notes if the behavior is not
obvious from the manifests.
