# Orbit shared workflows

The one build pipeline for every project deployed on [Orbit](https://yotam.xyz). App repos
call it and hold no CI logic of their own, so improving the build — caching, scanning, a
different registry — is one commit here that every project inherits.

## Why this repository is public, and why that is safe

It exists because a **public** repository cannot use a reusable workflow from a **private**
one; GitHub only allows public → public. The platform repository is private and stays that
way, so the shared pipeline lives here instead. Making the platform repository public to
solve a CI problem would have exposed its whole infrastructure configuration.

Nothing here is secret, and as of 2026-08-30 nothing here takes a secret either. This
repository stores no credentials and has no access to any cluster.

Anyone may call it. Doing so builds an image into *their own* registry namespace — and that
is all it does. It used to also post a `repository_dispatch` to the Orbit platform using an
`ORBIT_DISPATCH_TOKEN` the caller supplied; the platform now reads the build event through a
GitHub App it installs on the repositories it deploys, so the caller needs no token and this
pipeline no longer talks to the platform at all. Being able to run the pipeline was never
authorization to deploy, and now it is not even a way to ask.

## Usage

```yaml
name: build
on:
  push:
    branches: [main]

# Required. The called workflow needs packages: write to push to GHCR, and a called
# workflow can only narrow the caller's token, never widen it — a caller that omits this
# fails at startup with no jobs and no log line explaining why.
permissions:
  contents: read
  packages: write

jobs:
  build:
    uses: yotam-personal/workflows/.github/workflows/build-service.yml@main
    with:
      image: my-org/my-service         # GHCR repository, without the registry host
      context: .                       # optional, default "."
      dockerfile: Dockerfile           # optional, relative to context
      test: "make test"                # optional; runs before anything is pushed
```

No `secrets:` block: the pipeline accepts none. Passing one is now an error.

`@main` rather than a pinned SHA is deliberate: instant propagation is the entire point of a
shared pipeline, and both ends of this are owned by the same person. A third party consuming
it should pin a tag.
