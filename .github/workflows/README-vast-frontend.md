# vast-frontend reusable lane

`workflow_call` workflow for building sasasasa-family frontends on
**untrusted rented vast.ai runners** and handing the artifact to a
trusted `abar` ship job — with zero secrets on the rented side.

Spec: `docs/vast-frontend-lanes.md` in `ogiste/ctrl-plane`
("The standard lane — one reusable workflow").

## Trust boundary

- No `secrets:` inputs exist. Never smuggle env blobs, SSH keys, or
  registry creds through `install-cmd`/`build-cmd`.
- Public build-time env (`NEXT_PUBLIC_*`, `VITE_*`) goes in GitHub
  **variables** (`vars.*`) on the caller repo — never in `*_FRONTEND_ENV`
  style secret blobs that mix public and private values.
- The artifact is the payload. Every job ends with a workspace wipe.
- Job names start with `frontend-` because the lane allowlist globs on
  job names (`repo:frontend-*` in `vast-runner-repos.conf`).

## Contract

Inputs: `install-cmd`*, `build-cmd`*, `artifact-path`*,
`e2e-port` (number, 0=off), `env-scope` (`staging`|`prod`, default
`staging`), `node-version`, `working-directory` (default `.`),
`artifact-suffix` (for dual-lane runs).

Outputs: `artifact-name`, `manifest-sha256` (digest of the normalized
per-file manifest uploaded alongside the artifact).

## Caller example (~15 lines)

```yaml
jobs:
  probe:                            # is a vast runner up?
    runs-on: [self-hosted, vast]
    steps: [{ run: echo probe-ok }]
  build:
    needs: probe
    uses: Dreamscape-Mastermind/.github/.github/workflows/vast-frontend.yml@main
    with:
      install-cmd: corepack pnpm install --frozen-lockfile
      build-cmd: pnpm build
      artifact-path: dist/
      env-scope: staging
  ship:                             # trusted half — keep your own
    needs: build
    runs-on: [self-hosted, abar]
    steps:
      - uses: actions/download-artifact@v4
        with: { name: ${{ needs.build.outputs.artifact-name }} }
      - run: rsync -a dist/ deploy@box:/srv/app && pm2 reload app
```

## env-scope=prod — dual-lane digest compare

v1 ships the **single-lane** contract only. Until a dual-lane variant
exists, prod callers should:

1. Invoke this workflow **twice** with distinct `artifact-suffix`
   values (e.g. `lane-a`, `lane-b`) — the two jobs land on different
   rented hosts.
2. In the abar ship job, compare the two `manifest-sha256` outputs;
   on mismatch, refuse to ship and alert. A poisoner must own two
   randomly-assigned hosts in one window.
3. Keep the post-deploy tripwire: after rsync + pm2 reload, hash the
   *served* bundle and compare to `manifest-sha256`.
