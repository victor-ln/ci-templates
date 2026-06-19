# dtlk-ci-templates

CI/CD **reutilizável** da Datawake: *reusable workflows* (`workflow_call`) + *composite actions* por stack,
consumidos por qualquer repo da organização.

> Repo privado: habilitar **Settings → Actions → General → Access → "Accessible from repositories owned by
> Datawake-tec"** para os outros repos poderem usar.

## Estrutura

GitHub exige que *reusable workflows* fiquem **flat** em `.github/workflows/` (subpastas não são suportadas
para o arquivo chamado). Por isso os **gates por stack** ficam flat e a lógica reutilizável mora em **pastas
reais** sob `actions/<stack>/` (composite actions).

```
.github/workflows/        # reusable workflows (workflow_call) — FLAT
  ci-dotnet.yml           # test (unit+integração) + format + docker build
  ci-node.yml             # lint + test + docker build (vite/next/express)
  ci-python.yml           # ruff+bandit + pytest + docker build (fastapi)
  ci-docker.yml           # só docker build (repos docker-only)
  build-push.yml          # build + push no Harbor
  promote.yml             # re-tag do digest (build-once + promote)
  release.yml             # semantic-release (.releaserc padronizado aqui)
  gitops-bump.yml         # bump do newTag no dtlk-gitops via PR auto-aprovado
actions/                  # composite actions — PASTAS reais por stack
  dotnet/{test,format}/   node/{lint,test}/   python/{lint,test}/   docker/build/
```

## Gates de qualidade (PR) — por stack

| Workflow | Faz | Inputs principais |
|---|---|---|
| `ci-dotnet.yml` | `dotnet test` (unit+integração, coverage) + `dotnet-format` + docker build | `solution`, `dotnet-version` (`5.0.x`), `test_filter`, `run_format` (`true`), `build_args` |
| `ci-node.yml` | `npm run <lint>` + `npm test --coverage` + docker build | `node-version` (`20`), `prisma` (`false`), `run_tests` (`true`), `lint_script` (`lint`), `build_args` |
| `ci-python.yml` | `ruff check`/`format --check` + `bandit` + `pytest --cov` + docker build | `python-version` (`3.12`), `extras` (`dev`), `lint_paths` (`app/ tests/`), `package` (`app`), `test_path` (`tests/`), `run_tests` |
| `ci-docker.yml` | só `docker build` (sem push) | `context`, `dockerfile`, `build_args` |

Cada job faz checkout do repo **chamador** e usa as composite actions de `actions/<stack>/`. Nenhum gate dá push
nem precisa de secret.

## CD / GitOps

| Workflow | Faz | Inputs | Secrets |
|---|---|---|---|
| `build-push.yml` | build + push no Harbor (tags via metadata-action) | `project`, `app`, `tags`, `version` | `HARBOR_USERNAME`, `HARBOR_PASSWORD` |
| `promote.yml` | re-tag do digest (`HEAD^2`) → `vX.Y.Z`+`latest` (+ fallback rebuild). Output: `version` | `project`, `app` | `HARBOR_USERNAME`, `HARBOR_PASSWORD` |
| `release.yml` | semantic-release no repo chamador → tag `vX.Y.Z` (`.releaserc` **padronizado aqui**) | `release_branches` (`["main"]`) | `SEMANTIC_RELEASE_TOKEN` |
| `gitops-bump.yml` | atualiza `newTag` no `dtlk-gitops` e abre PR auto-aprovado | `project`, `app`, `env`, `new_tag` | `GITOPS_TOKEN` |

## Consumo — gate de PR (exemplos)

```yaml
# .NET (ex.: frm-api-dotnet) — .github/workflows/ci.yml
on: { pull_request: { branches: [develop, release, main] } }
jobs:
  ci:
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/ci-dotnet.yml@v1
    with: { solution: Simjob.Framework.sln, dotnet-version: "5.0.x", run_format: false }
```

```yaml
# Node (express com Prisma+testes)
jobs:
  ci:
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/ci-node.yml@v1
    with: { prisma: true }
# Vite/Next (sem suíte de testes, com build-arg)
#   uses: .../ci-node.yml@v1
#   with: { run_tests: false, build_args: "VITE_API_URL=https://staging-api.datawake.cloud" }
```

```yaml
# Python (FastAPI)
jobs:
  ci:
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/ci-python.yml@v1
```

## Consumo — CD GitFlow (build-once + promote)

```yaml
# deploy-dev.yml — push em `release` (validação) → :sha-<7> → bump overlay dev
on: { push: { branches: [release] } }
jobs:
  vars:
    runs-on: ubuntu-latest
    outputs: { sha7: "${{ steps.s.outputs.sha7 }}" }
    steps:
      - id: s
        run: echo "sha7=${GITHUB_SHA:0:7}" >> "$GITHUB_OUTPUT"
  build:
    needs: vars
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/build-push.yml@v1
    with:
      project: dw-framework
      app: frm-api-dotnet
      tags: |
        type=raw,value=sha-${{ needs.vars.outputs.sha7 }}
      version: 0.0.0-dev.${{ github.run_number }}
    secrets: inherit
  bump:
    needs: [vars, build]
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/gitops-bump.yml@v1
    with: { project: dw-framework, app: frm-api-dotnet, env: dev, new_tag: "sha-${{ needs.vars.outputs.sha7 }}" }
    secrets: inherit
```

```yaml
# release.yml — push em `main` → tag vX.Y.Z
on: { push: { branches: [main] } }
jobs:
  release:
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/release.yml@v1
    secrets: inherit

# promote-prod.yml — tag v* → promote do digest + bump overlay prod
on: { push: { tags: ["v*"] } }
jobs:
  promote:
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/promote.yml@v1
    with: { project: dw-framework, app: frm-api-dotnet }
    secrets: inherit
  bump:
    needs: promote
    uses: Datawake-tec/dtlk-ci-templates/.github/workflows/gitops-bump.yml@v1
    with: { project: dw-framework, app: frm-api-dotnet, env: prod, new_tag: "${{ needs.promote.outputs.version }}" }
    secrets: inherit
```

Repos **feat→prod** (sem `release`): pulam o `deploy-dev`; o build+bump de prod sai do `promote-prod`
(ou de um `build-push`+`gitops-bump` direto no overlay `prod`).

## Versionamento

Pin por tag: `@v1`. Ao mudar contrato, criar `v2` (e mover a tag `v1` para patches retrocompatíveis).
