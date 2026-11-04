# devops

Workflows reutilizáveis do GitHub Actions da Anext.

| Workflow | O que faz |
| --- | --- |
| [`docker-build-multi.yml`](.github/workflows/docker-build-multi.yml) | Faz o build de uma imagem Docker e o push para o Amazon ECR, o GitHub Container Registry ou um registry genérico. |
| [`security.yml`](.github/workflows/security.yml) | Roda o Trivy no repositório de quem chama, procurando vulnerabilidades, erros de configuração e segredos expostos. |

## Exemplo de uso

```yaml
name: CI

on:
  push:
    branches: [main]
  pull_request:

jobs:
  security:
    uses: anextbr/devops/.github/workflows/security.yml@<sha> # vX.Y.Z
    permissions:
      contents: read

  build:
    needs: security
    if: github.event_name == 'push'
    uses: anextbr/devops/.github/workflows/docker-build-multi.yml@<sha> # vX.Y.Z
    permissions:
      contents: read
      packages: write
    with:
      registryType: ecr
      dockerFile: Dockerfile
      dockerContext: .
      imageRepository: minha-aplicacao
      imageTagFrom: shaCommit
    secrets:
      AWS_ACCESS_KEY_ID: ${{ secrets.AWS_ACCESS_KEY_ID }}
      AWS_SECRET_ACCESS_KEY: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
      AWS_REGION: ${{ secrets.AWS_REGION }}
```

A imagem vai para `<conta>.dkr.ecr.<região>.amazonaws.com/minha-aplicacao:<sha-do-commit>`. Se o repositório ainda não existir no ECR, o workflow cria.

## `docker-build-multi.yml`

### Inputs

| Input | Obrigatório | Padrão | Descrição |
| --- | --- | --- | --- |
| `dockerFile` | sim | | Caminho do Dockerfile. |
| `dockerContext` | sim | | Contexto do build. |
| `registryType` | não | `generic` | `ecr`, `github` (ghcr.io) ou `generic`. |
| `imageRepository` | não | nome do repositório que chama, em minúsculas | Nome do repositório da imagem. |
| `imageTag` | não | | Tag fixa. Quando informada, `imageTagFrom` é ignorado. |
| `imageTagFrom` | não | `latest` | `shaCommit` (SHA do commit) ou `branchTagName` (nome da branch ou tag, com `/` trocado por `-`). |
| `registryServer` | não | `vars.REGISTRY_SERVER` | Host do registry. Só é usado com `registryType: generic`. |
| `registryNamespace` | não | `vars.REGISTRY_NAMESPACE` | Prefixo do caminho da imagem (`<registry>/<namespace>/<repositório>`). |

### Secrets

Os secrets precisam ser passados explicitamente. `secrets: inherit` só funciona dentro da mesma organização.

| Secret | Quando usar |
| --- | --- |
| `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION` | `registryType: ecr` |
| `REGISTRY_LOGIN`, `REGISTRY_PASSWORD` | `registryType: generic` |
| `GCP_SVC_DEVOPS_JSON_KEY` | Opcional: cópia da imagem para o GCP Artifact Registry (veja abaixo). |

Com `registryType: github` não é preciso passar nada: o workflow usa o `GITHUB_TOKEN`.

### Variables (opcionais)

As variables são lidas do repositório ou da organização de quem chama.

| Variable | Efeito |
| --- | --- |
| `REGISTRY_SERVER` | Valor padrão de `registryServer`. |
| `REGISTRY_NAMESPACE` | Valor padrão de `registryNamespace`. |
| `GCP_ARTIFACTORY_BASE_PATH` | Com `registryType: ecr`, também envia a imagem para esse caminho do GCP Artifact Registry. Exige o secret `GCP_SVC_DEVOPS_JSON_KEY`. |

### Saída

O job publica um artifact chamado `build`, guardado por 7 dias. Ele contém:

- `stage.build.out`, com `IMAGE_REPOSITORY`, `IMAGE_TAG` e `IMAGE_NAME` (e `IMAGE_REPOSITORY_GCP`, quando a cópia para a GCP está ativa);
- uma cópia da pasta `.k8s/` do repositório que chama, se ela existir.

> Em repositórios públicos, qualquer pessoa pode baixar esse artifact. Não deixe segredos em texto puro nos manifests de `.k8s/`.

## `security.yml`

Não tem inputs nem secrets. Ele faz o checkout do repositório que chama e mostra no log os achados de severidade `MEDIUM` ou mais alta. Por enquanto o job não falha quando encontra algum problema.

## Boas práticas

- **Fixe a versão pelo SHA do commit** (`@<sha>`) em vez de `@main`. Assim uma mudança aqui não afeta todos os repositórios de uma vez.
- **Conceda as permissões do exemplo** no job que chama (`contents: read` e `packages: write`). O `docker-build-multi.yml` pede `packages: write` para conseguir enviar ao ghcr.io, e o GitHub não deixa um workflow chamado pedir mais permissões do que quem chama concede. Por isso o job falha se essa permissão faltar, mesmo quando o registry é o ECR.
