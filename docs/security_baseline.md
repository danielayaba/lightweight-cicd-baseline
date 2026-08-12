# Security baseline checklist

Documented security baseline for the lightweight CI/CD pipeline. As stated in the
research proposal, security is handled as a documented reference checklist rather
than enforced gates, since building and testing full security controls would
constitute a project in its own right.

Each item below was checked against the repository on 3 August 2026. Items that do not
hold are left unticked and explained rather than quietly claimed; they are collected
again under "Open items" at the end.

## Secrets and credentials
- [x] No credentials, tokens, or API keys are hard-coded anywhere in the repository.
- [x] All deployment tokens are stored in GitHub encrypted secrets.
- [x] The Render deploy hook URL is stored as the `RENDER_DEPLOY_HOOK_URL` secret. It is
  itself a credential — anyone holding it can trigger a deployment — so it is rotated
  from Render → Settings → Build & Deploy → Deploy Hook if it is ever exposed.
- [x] Secrets are never echoed to workflow logs. GitHub additionally masks registered
  secret values if one is printed by accident, so the two protections are independent.

## Configuration and environment variables
- [x] `RENDER_SERVICE_URL` is stored as a GitHub repository variable and is likewise
  never echoed to workflow logs: only its name appears in the workflow's diagnostic
  messages, never its value. It is a variable rather than a secret because the service
  URL is public information and grants no capability — unlike the deploy hook, it can
  only be read from, not deployed with.
- [x] The distinction matters operationally: GitHub masks secrets in logs but does not
  mask variables, so keeping `RENDER_SERVICE_URL` out of the output is a property of how
  the workflow is written, not something the platform enforces.
- [x] Every other environment variable the application needs at runtime is entered in
  Render as a service environment variable for the deployment, rather than baked into
  the image or committed to the repository. Two values are already handled this way
  without any manual entry: `PORT` is injected by Render and read by the application,
  and `NODE_ENV=production` is set in the Dockerfile.

## Container
- [x] Base image is a minimal image (`node:20-alpine`) — see the open item below on its
  support status.
- [x] Container runs as a non-root user (`USER node`).
- [x] Multi-stage build keeps build tooling out of the runtime image.
- [x] `.dockerignore` excludes `node_modules`, `.git`, `.github`, `docs`, `test`,
  markdown files and `.gitignore` from the build context, so neither the git history nor
  the project documentation reaches the image.
- [ ] Base image is on a supported release line. **It is not**: Node 20 reached end of
  life on 30 April 2026 and no longer receives security updates, so vulnerabilities
  found in it from that date onwards will not be patched.

## Workflow permissions
- [x] The workflow starts read-only (`contents: read`) and widens per job only where
  needed: `packages: write` is granted to `containerise-deploy` alone, for the registry
  push. No other job can write anything.
- [x] The default `GITHUB_TOKEN` is used for registry authentication rather than a
  personal access token, so the credential is scoped to the run and expires with it.

## Dependencies
- [x] `package-lock.json` is committed, so `npm ci` performs a reproducible install from
  pinned versions rather than re-resolving the dependency tree on each run.
- [x] The production image installs production dependencies only (`--omit=dev`), so the
  linter and its transitive dependencies never reach the runtime image.

## Licensing and attribution
- [x] All third-party and open-source code fragments are attributed. The project has a
  single development dependency (ESLint) and no vendored third-party code.
- [ ] Project is published under the MIT License. **Declared but not complete**: MIT is
  stated in `package.json` and the README, but no `LICENSE` file exists at the
  repository root, so the licence terms are asserted without being reproduced.

## Open items

Two items above do not currently hold. Both are recorded here rather than presented as
satisfied, since a baseline that ticks boxes it has not met is worth less than one that
names its gaps.

1. **End-of-life base image.** `node:20-alpine` is minimal, but Node 20 went out of
   support on 30 April 2026. Moving to `node:22-alpine` (supported to April 2027) or
   `node:24-alpine` (April 2028) is a two-line change in the Dockerfile. It is deferred
   deliberately: changing the base image alters build times and image contents, and
   doing so partway through a benchmarking series would make the runs incomparable. It
   should be applied between series, not during one.
2. **Missing LICENSE file.** Adding the MIT licence text at the repository root would
   close the gap.
