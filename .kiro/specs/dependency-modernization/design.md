# Dependency Modernization — Design

## Goal

Update the Amazon GameLift Testing Toolkit dependencies to supported,
current versions **without breaking the deploy pipeline or runtime
behaviour**. Each phase is independently verifiable (build + synth +
deploy) so work can pause and resume safely across sessions.

Non-goal: Feature changes, UI redesigns, or CloudFormation-affecting
refactors beyond what a dependency bump forces.

## Repository layout & git workflow

### Remotes
- `origin` → `https://github.com/fastshow/amazon-gamelift-testing-toolkit.git` (work target)
- `upstream` → `https://github.com/aws-samples/amazon-gamelift-testing-toolkit.git` (read-only, push disabled)

### Branches
- `main` — pristine, tracks `upstream/main`. Never commit modernization work here. Used to `git pull upstream main` and rebase downstream branches as needed.
- `modernize` — long-lived integration branch. Each completed phase is merged in with a `--no-ff` merge commit so phase boundaries stay visible.
- `phase-*` — per-phase working branches cut from `modernize`, deleted after merge.

### Tags
- `v1.0.0-pre-modernize` — upstream HEAD at the start of the work (`b31255a`). Reference point for full-delta diffs.
- Tag each phase on merge, e.g. `phase-1-complete`.

### Commit conventions
- `chore(scope):`, `fix(scope):`, `feat(scope):` — keeps phases legible and makes changelogs trivial.
- Phase-specific commits land on `phase-*` branches; merges into `modernize` use `--no-ff` so the branch boundary is preserved.

## Verification gates

Every phase must clear these before merging into `modernize`:

1. `yarn build-toolkit` exits 0.
2. `cd ManagementConsole/Infra && npx cdk synth` exits 0 and produces a `cdk.out` assembly.
3. `yarn deploy-toolkit` succeeds in the test AWS account (eu-central-1) with the Management Console Lambdas reaching `ACTIVE` state and serving a basic request.
4. Deprecation warnings tracked, but do not block merge unless they indicate an imminent breakage.
5. No new secrets, credentials, or AWS account identifiers committed.

Sample-game-specific phases additionally require `yarn build-sample-game` and `yarn deploy-sample-game` to succeed.

## Phase map

| Phase | Scope | Verification |
|-------|-------|--------------|
| 1 | Toolkit .NET 6 → .NET 8, CDK 2.124 → 2.178 | build + synth + deploy (incl. Lambda init) |
| 2 | Toolkit CDK 2.178 → 2.253.x (latest stable), Cognito IdentityPool alpha → stable, deprecation cleanup | build + synth + deploy |
| 3 | Toolkit AWS SDK + Amazon.Lambda.* patch/minor bumps | build + synth + deploy |
| 4 | UI dependency bumps (typescript 5, webpack-cli 5, webpack-dev-server 5, @types/node 20, babel-loader 9, copy-webpack-plugin 12). Replace `aws-sdk-js-v3` GitHub tarball with explicit `@aws-sdk/client-*` packages. Clean up `ManagementConsole/UI/static/package.json`. | build + synth + deploy + smoke-test web UI in CloudFront |
| 5 | UI Amplify 4 → 6 migration (auth API surface changes) | build + synth + deploy + manual login |
| 6 | Sample game: .NET 6 → .NET 8, CDK bump, Dockerfile `dotnet-sdk-7.0` → `dotnet-sdk-8.0`, fix `cp` path for net8 | build-sample-game + deploy-sample-game + join a match |
| 7 | GitHub Actions: pin versions (`actions/checkout@v4`, `setup-node@v4`, `setup-dotnet@v4`); docs Dockerfile python:3.9 → current python:3.12-alpine | Actions run green |
| 8 | Docs accuracy — README + quick_start refresh (.NET 8 everywhere, Node 18+, correct deploy sequence) | `host-docs-website-local` renders |

Phases are ordered so each bump has the minimum viable ecosystem to land. Phase 5 intentionally follows Phase 4 because Amplify v6 touches UI source, not just `package.json`.

## Known fragilities / risks

- **CDK CLI / lib version divergence (CDK 2.179+).** After Phase 2 we will track the CLI `2.10xx` series independently from the lib. `yarn bootstrap` against an older CDK-bootstrap stack may trigger an in-place upgrade; acceptable, non-reversible at the AWS account level.
- **Cognito IdentityPool alpha → stable migration.** API surface is similar but not identical. Phase 2 must preserve auth/unauth role ARN outputs consumed by the UI. Full end-to-end login test is the acceptance gate.
- **UI `aws-sdk-js-v3` GitHub tarball.** Currently references the entire `aws/aws-sdk-js-v3` monorepo; replacement with specific client packages in Phase 4 may slightly change bundle size and import paths.
- **Amplify v4 → v6.** Breaking changes in `Auth` API. Defer to Phase 5 so it's isolated.
- **.NET 10 SDK also installed.** `global.json` pins 8.0.420 to prevent accidental use of net10-era behaviour until intentionally bumped.
- **Stale build outputs.** Bumping the TFM without cleaning `bin/Release/net6.0` caused Phase 1's production outage (CDK packaged the old folder). Every phase that touches the TFM or output paths must wipe the old directory as part of the fix.

## Tooling state on the dev machine (for resumption)

- macOS, zsh.
- `dotnet` 8.0.420 via `brew install --cask dotnet-sdk@8`. (10.0.203 also installed; `global.json` pins 8.)
- `dotnet-lambda` 6.0.5 installed as a global tool at `~/.dotnet/tools`. Add to PATH in any shell: `export PATH="$PATH:/Users/cbp/.dotnet/tools"`.
- `yarn` 1.22.22 via `npm install -g yarn`.
- `node` 22.16.0 (warning only; jsii doesn't officially support 22, passes in practice).
- Docker not installed. Needed for Phase 6 (sample game).
- AWS credentials: user exports in their interactive terminal; Kiro tool shells do not inherit them. Deploys must be run by the user in their terminal.
- Target region: `eu-central-1`.
