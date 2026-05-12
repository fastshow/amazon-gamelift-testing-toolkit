# Dependency Modernization — Tasks

Persistent to-do list across sessions. Check off as each item lands on
`modernize`. Each top-level phase corresponds to one `phase-*` branch.

## Phase 1 — Toolkit .NET 8 + CDK 2.178

- [x] Pin .NET SDK via `global.json` (8.0.420)
- [x] `ManagementConsoleBackend.csproj`: net6.0 → net8.0
- [x] `ManagementConsoleInfra.csproj`: net6.0 → net8.0
- [x] `aws-lambda-tools-defaults.json`: net6.0/dotnetcore3.1 → net8.0/dotnet8
- [x] Infra `Program.cs`: `Runtime.DOTNET_6` → `Runtime.DOTNET_8`
- [x] `Amazon.CDK.Lib`: 2.124.0 → 2.178.2
- [x] `Amazon.CDK.AWS.Cognito.IdentityPool.Alpha`: 2.38.1-alpha.0 → 2.178.2-alpha.0
- [x] Root `package.json`: `aws-cdk` CLI 2.124.0 → 2.178.2
- [x] `BackendStack.cs`: add `CfnIntegration` / `CfnIntegrationProps` using aliases (resolve new `Amazon.CDK.AWS.Logs.CfnIntegration` collision)
- [x] `BackendStack.cs` + `WebStack.cs`: `CodeRoot` path `/bin/Release/net6.0` → `/bin/Release/net8.0` (hotfix for deployed Lambdas failing runtime init)
- [x] Verify: `yarn build-toolkit` green
- [x] Verify: `cdk synth` green (deprecation warnings OK)
- [x] Verify: `yarn deploy-toolkit` succeeds and Lambdas reach ACTIVE state

Merged into `modernize` as `ec06e22`, merge commit `fc5a8a7`.

## Phase 2 — CDK to latest stable + IdentityPool alpha → stable + deprecation cleanup

- [ ] Bump `Amazon.CDK.Lib` to latest stable 2.253.x
- [ ] Bump `aws-cdk` CLI to matching 2.10xx.x (CLI/lib divergence begins here)
- [ ] Bump `Constructs` to 10.4.x
- [ ] Bump `Cdklabs.CdkNag` to current
- [ ] Remove `Amazon.CDK.AWS.Cognito.IdentityPool.Alpha` package reference
- [ ] `WebStack.cs`: migrate `IdentityPool` / `UserPoolAuthenticationProvider` / `IdentityPoolAuthenticationProviders` from alpha namespace to stable `Amazon.CDK.AWS.Cognito` API
- [ ] `WebStack.cs`: replace deprecated `CloudFrontWebDistribution` with `Distribution` (S3Origin, OriginAccessIdentity / OriginAccessControl, ViewerCertificate → ViewerCertificate builder)
- [ ] `DataStack.cs`: replace all `TableProps.PointInTimeRecovery` (17 sites) with `PointInTimeRecoverySpecification`
- [ ] `yarn bootstrap` — in-place CDK bootstrap stack upgrade; call out to user before running
- [ ] Verify: `yarn build-toolkit` green, warning count drops
- [ ] Verify: `cdk synth` green; CloudFormation diff reviewed for behavioural changes (e.g. OAI → OAC migration may cause CloudFront replacement; requires planning)
- [ ] Verify: `yarn deploy-toolkit` succeeds; confirm Cognito authenticated and unauthenticated role ARNs match previous stack outputs (the UI reads them from `Config.json`)
- [ ] Verify: manual login via Cognito user works end-to-end

## Phase 3 — AWS SDK / Amazon.Lambda patch bumps

- [ ] Enumerate current vs latest for all `AWSSDK.*` packages in `ManagementConsoleBackend.csproj`
- [ ] Enumerate current vs latest for all `Amazon.Lambda.*` packages; stay on `Amazon.Lambda.Serialization.SystemTextJson` 2.x (3.x changes JSON defaults)
- [ ] Bump to latest 3.7.x patches
- [ ] Verify: build + synth + deploy
- [ ] Verify: state poller Step Function still runs a poll cycle successfully

## Phase 4 — UI dependency bumps (conservative)

- [ ] `@types/node` 14 → 20
- [ ] `typescript` 4 → 5 (expect minor type fixes)
- [ ] `webpack-cli` 4 → 5
- [ ] `webpack-dev-server` 4 → 5
- [ ] `babel-loader` 8 → 9
- [ ] `copy-webpack-plugin` 9 → 12
- [ ] `clean-webpack-plugin` 4.0.0-alpha.0 → 4.0.0 (stable)
- [ ] `jsoneditor` 9 → 10 (check API)
- [ ] Replace `aws-sdk-js-v3: "aws/aws-sdk-js-v3"` GitHub tarball with explicit `@aws-sdk/client-*` packages actually used
- [ ] Set `engines.node` from `>=14` to `>=18`
- [ ] Clean up `ManagementConsole/UI/static/package.json`: remove `install` and `npm` deps, evaluate `mdbootstrap` upgrade path
- [ ] Verify: `yarn build-web` green; bundle loads in browser
- [ ] Verify: end-to-end deploy + CloudFront-served UI still renders and connects to the WebSocket API

## Phase 5 — UI Amplify 4 → 6

- [ ] Replace `@aws-amplify/auth` 4.x with `aws-amplify` 6.x + `@aws-amplify/auth` v6
- [ ] Update all UI call sites: `Auth.signIn` → `signIn`, etc. (named imports, not default)
- [ ] Update `Amplify.configure` to v6 shape (`Auth.Cognito.userPoolId` / `userPoolClientId`)
- [ ] Verify: full login flow works, session persisted, refresh token path intact

## Phase 6 — Sample game modernization

- [ ] `SampleGameInfra.csproj`: net6.0 → net8.0
- [ ] `SampleGameBackend.csproj`: net6.0 → net8.0
- [ ] `SampleGameBuild.csproj`: net6.0 → net8.0
- [ ] `SampleGame/Infra/src/Program.cs`: `Runtime.DOTNET_6` → `Runtime.DOTNET_8`
- [ ] Sample game CDK deps: match Phase 2 toolkit versions (`Amazon.CDK.Lib` latest stable, drop alpha IdentityPool)
- [ ] `SampleGame/Backend/aws-lambda-tools-defaults.json`: runtime strings
- [ ] `SampleGame/Game/Dockerfile`: `dotnet-sdk-7.0` → `dotnet-sdk-8.0`
- [ ] `SampleGame/Game/Dockerfile`: update `cp src/GameLiftServerSDK/bin/x64/Release/net6.0/*` path to `net8.0`
- [ ] Check hardcoded `SampleGame/.../bin/Release/net6.0` CodeRoot paths (mirror of the Phase 1 hotfix)
- [ ] Clean stale `bin/Release/net6.0` + `obj/Release/net6.0` directories under `SampleGame/`
- [ ] Install Docker on dev machine
- [ ] Verify: `yarn build-sample-game` green
- [ ] Verify: `yarn deploy-sample-game` succeeds
- [ ] Verify: GameLift fleet reaches `ACTIVE`, virtual player joins a match

## Phase 7 — CI & docs infra

- [ ] `.github/workflows/build.yml`: `actions/checkout@v3` → `@v4`, `actions/setup-node@v3` → `@v4`, `actions/setup-dotnet@v3` → `@v4`, dotnet-version to 8
- [ ] `.github/workflows/docs.yml`: same action bumps
- [ ] `.github/workflows/release-prep.yml`: same action bumps
- [ ] `docs/docs/Dockerfile`: `python:3.9.2-alpine3.13` → `python:3.12-alpine3.19` (or latest)
- [ ] Verify: `yarn host-docs-website-local` still serves
- [ ] Verify: GitHub Actions run green on `modernize`

## Phase 8 — Docs accuracy pass

- [ ] `README.md`: prereqs .NET 6 → .NET 8, Node 18.x confirmed
- [ ] `docs/docs/quick_start.md`: prereqs .NET 5 → .NET 8, Node 16.x → 18.x (currently badly out of date)
- [ ] Add a "supported toolchain" steering note to `.kiro/steering/`
- [ ] Verify: docs site renders without broken links

## Ongoing / parking lot

- [ ] Decide whether to enforce `--no-ff` via hook or steering rule
- [ ] Consider enabling Dependabot on the fork once Phase 4 lands
- [ ] After all phases: rebase `modernize` onto latest `upstream/main`, resolve any conflicts, tag `v2.0.0-modernized`
