# DevOps — Anext

Olá! 👋 Seja bem-vindo(a) ao repositório de DevOps da Anext.

## Propósito

Este repositório centraliza os **workflows reutilizáveis do GitHub Actions** usados pelos projetos da Anext, para que build, publicação de imagens e verificações de segurança sigam o mesmo padrão em todos os repositórios, sem duplicar configuração.

## Workflows disponíveis

| Workflow | O que faz |
| --- | --- |
| [docker-build-multi.yml](.github/workflows/docker-build-multi.yml) | Faz o build e o push de imagens Docker para OCIR (padrão), GitHub Container Registry ou AWS ECR (com cópia para o GCP Artifact Registry), e publica um artefato `build` com as informações da imagem gerada. |
| [security.yml](.github/workflows/security.yml) | Roda o Trivy em pull requests para `main`, `master` e `develop`, procurando vulnerabilidades, problemas de configuração e segredos expostos. |

## Exemplo de uso

```yaml
jobs:
  build:
    uses: anextbr/devops/.github/workflows/docker-build-multi.yml@main
    with:
      dockerFile: Dockerfile
      dockerContext: .
      imageTagFrom: branchTagName
    secrets: inherit
```
