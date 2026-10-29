# codecube-workflows

Reusable GitHub Actions workflows and actions for the `CodeCube.*` NuGet packages.

| Item | Purpose | Used from |
|---|---|---|
| `.github/workflows/dotnet-ci.yml` | restore, build, test a solution | `ci.yml` in each repo (push / pull request) |
| `.github/workflows/nuget-pack.yml` | build, test, pack one csproj; uploads artifact `nupkg` | `release.yml` in each repo (tag `vX.Y.Z` or `<Project>/vX.Y.Z`) |
| `.github/actions/nuget-publish` | composite action: NuGet Trusted Publishing (OIDC) + `dotnet nuget push` | `publish` job in each repo's `release.yml` |

## Why the publish step is not a reusable workflow

nuget.org Trusted Publishing validates the OIDC claim `job_workflow_ref`, which for a reusable
workflow points at *this* repository. The policy on nuget.org is registered per package repository,
so the job that calls `NuGet/login` must live in the package repository's own `release.yml`.
A composite action runs inside that job, so it keeps the correct claims.
See <https://github.com/NuGet/NuGetGallery/issues/11000>.

## Versioning

Callers reference these workflows by the moving tag `v1`
(`roblohmann/codecube-workflows/.github/workflows/nuget-pack.yml@v1`,
`roblohmann/codecube-workflows/.github/actions/nuget-publish@v1`).
Breaking changes to inputs get a new major tag (`v2`); compatible changes move `v1` forward:

```bash
git tag -f v1 && git push -f origin v1
```

## Release flow in a package repository

1. Set `<VersionPrefix>` in `Directory.Build.props` (single-package repo) or in the csproj (multi-package repo).
2. Commit, then push a tag: `git tag v9.0.5 && git push origin v9.0.5`
   (multi-package repos: `Versioning/v8.0.10`).
3. `release.yml` checks the tag against `VersionPrefix`, calls `nuget-pack.yml` (builds with
   `-p:Version=<tag>`, packs, uploads artifact) and then runs the `publish` job in environment
   `nuget.org`, which exchanges the GitHub OIDC token for a one-hour nuget.org API key and pushes.

## Requirements per calling repository

- A Trusted Publishing policy on <https://www.nuget.org/account/trustedpublishing>:
  owner `roblohmann`, repository `<repo name>`, workflow file `release.yml`, environment `nuget.org`.
- GitHub environment `nuget.org` (no secrets needed).
- `publish` job with `permissions: id-token: write` and `contents: read`.
