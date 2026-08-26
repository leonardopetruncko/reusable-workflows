# reusable-workflows — ToggleMaster DevSecOps

Repositório central de **GitHub Actions reusable workflows** para os 5
microsserviços do ToggleMaster. Cada microsserviço só precisa de um
`.github/workflows/ci.yml` curto que *chama* estes workflows — assim, uma
correção ou melhoria de pipeline (ex.: trocar de Bandit para outra ferramenta)
é feita **uma vez aqui** e vale para os 5 repositórios.

## Workflows disponíveis

| Arquivo | Linguagem/uso | O que faz |
|---|---|---|
| `python-quality.yml` | Python | Lint (ruff + flake8) e Build & Unit Test (pytest + coverage) |
| `go-quality.yml` | Go | Lint (golangci-lint) e Build & Unit Test (go build + go test) |
| `security-python.yml` | Python | SAST (Bandit) + SCA (Trivy fs). Falha em `CRITICAL` |
| `security-go.yml` | Go | SAST (gosec) + SCA (Trivy fs). Falha em `CRITICAL` |
| `docker-build-push.yml` | Docker/ECR | Build, Container Scan (Trivy), login e push no ECR com tag `v1.0.0-<commit-hash>` |
| `update-gitops.yml` | GitOps | Atualiza a tag da imagem no `deployment.yaml` do repositório de GitOps e faz commit+push |

## Como usar em cada microsserviço

Exemplo para um serviço **Python** (`flag-service`, `targeting-service`, `analytics-service`):

```yaml
name: CI/CD - flag-service

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

permissions:
  id-token: write
  contents: read

jobs:
  quality:
    uses: 3DCLT-Fiap-Devops/reusable-workflows/.github/workflows/python-quality.yml@main
    with:
      python-version: '3.11'

  security:
    needs: quality
    uses: 3DCLT-Fiap-Devops/reusable-workflows/.github/workflows/security-python.yml@main
    with:
      trivy-severity: 'CRITICAL'

  docker:
    needs: security
    if: github.event_name == 'push'
    uses: 3DCLT-Fiap-Devops/reusable-workflows/.github/workflows/docker-build-push.yml@main
    with:
      dockerfile-path: 'Dockerfile.flag'
      image-name: 'flag-service'
      push-image: true
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}

  update-gitops:
    needs: docker
    if: github.event_name == 'push'
    uses: 3DCLT-Fiap-Devops/reusable-workflows/.github/workflows/update-gitops.yml@main
    with:
      gitops-repo: '3DCLT-Fiap-Devops/toogle-master-gitops'
      service-name: 'flag-service'
      image-uri: ${{ needs.docker.outputs.image-uri }}
      manifest-path: 'apps/flag-service/deployment.yaml'
    secrets:
      GITOPS_PUSH_TOKEN: ${{ secrets.GITOPS_PUSH_TOKEN }}
```

Para serviços **Go** (`auth-service`, `evaluation-service`), troque
`python-quality.yml`/`security-python.yml` por `go-quality.yml`/`security-go.yml`.

## Secrets necessários (configurar em cada repositório de microsserviço, ou como Org Secrets)

- `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY`: credenciais com permissão de push no ECR (idealmente um usuário/role dedicado só a isso).
- `GITOPS_PUSH_TOKEN`: Personal Access Token (fine-grained) com permissão de `contents:write` **apenas** no repositório de GitOps.

> Alternativa mais segura (evita chave estática de longa duração): configurar
> **OIDC** entre o GitHub Actions e o IAM (role assumida via
> `aws-actions/configure-aws-credentials` com `role-to-assume`). Não incluído
> aqui por causa das restrições do AWS Academy (não podemos criar roles/policies
> de IAM via Terraform), mas é a recomendação para a conta pessoal.

## Regra de bloqueio (DevSecOps)

Tanto `security-python.yml`/`security-go.yml` (SAST/SCA) quanto o Container
Scan em `docker-build-push.yml` usam `exit-code: '1'` do Trivy com
`severity: CRITICAL` por padrão — ou seja, **qualquer vulnerabilidade crítica
derruba o pipeline antes do push da imagem**, conforme exigido no enunciado.
