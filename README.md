# volundr-deploy

Deploy de imagens num servidor com o **Volundr**, o agente do
[Heimdall](https://github.com/dlduarte/heimdall-releases), a partir do CI: o
GitHub Actions, o GitLab CI e o Jenkins.

O servidor só troca a **imagem** de **serviços listados** num alvo, de
**repositórios listados**, e confere quem pede pelas regras de confiança do
alvo. Com OIDC, nenhum segredo fica guardado no CI. O job acompanha o log e
falha quando o deploy falha, inclusive quando o healthcheck falha e o
Volundr volta às imagens de antes.

O alvo, as regras e a entrada HTTPS são configurados no Heimdall, na seção
**Deploy** do servidor.

## GitHub Actions

```yaml
jobs:
  deploy:
    runs-on: ubuntu-latest
    environment: producao            # o ambiente da regra do alvo: a aprovação fica aqui
    permissions:
      id-token: write                # o token OIDC do job
    steps:
      - uses: dlduarte/volundr-deploy@v1
        with:
          host: https://deploy.loja.com.br
          target: loja
          images: |
            api=ghcr.io/acme/loja-api@${{ needs.build.outputs.digest }}
            worker=ghcr.io/acme/loja-api@${{ needs.build.outputs.digest }}
          reason: ${{ github.ref_name }}
```

| Entrada | |
| --- | --- |
| `host` | a URL da entrada do CI do Volundr (o `VOLUNDR_CI_URL` do servidor) |
| `target` | o alvo |
| `images` | um `serviço=imagem` por linha, por digest ou por tag |
| `reason` | a versão, o motivo: aparece no histórico |
| `rollback` | `true` volta às imagens de antes do último deploy |
| `timeout` | segundos esperando o fim (1800) |
| `token` | um token de deploy (`vd_…`) de um secret, no lugar do OIDC |
| `version` | a versão do `volundr` (`latest`) |

A saída `result` é o código do deploy.

## GitLab CI

```yaml
include:
  - remote: https://raw.githubusercontent.com/dlduarte/volundr-deploy/v1/gitlab/volundr-deploy.yml

deploy:
  extends: .volundr-deploy
  stage: deploy
  environment: producao
  variables:
    VOLUNDR_HOST: https://deploy.loja.com.br
    VOLUNDR_TARGET: loja
    VOLUNDR_IMAGES: api=$CI_REGISTRY_IMAGE@$DIGEST
  id_tokens:
    VOLUNDR_TOKEN:
      aud: https://deploy.loja.com.br
```

Veja [`gitlab/volundr-deploy.yml`](gitlab/volundr-deploy.yml).

## Jenkins

Com um token de deploy numa credencial (`withCredentials`), ou pelo **portão
SSH** — a chave do alvo com o `sshagent`, sem porta nova no servidor. Veja
[`jenkins/Jenkinsfile`](jenkins/Jenkinsfile).

## Em qualquer lugar

O `volundr` é um binário só, para Linux, macOS e Windows, publicado em cada
[release do Heimdall](https://github.com/dlduarte/heimdall-releases/releases)
com o `volundr-SHA256SUMS`:

```bash
volundr deploy --host https://deploy.loja.com.br --target loja \
  --image api=ghcr.io/acme/loja-api@sha256:… --reason v2.4.1
```

O token vem de `VOLUNDR_TOKEN` (um token de deploy, ou o `id_token` do
GitLab) ou, no GitHub Actions, do próprio job.

| Saída | Significa |
| --- | --- |
| 0 | concluído |
| 1 | falhou, e o alvo ficou como estava (revertido, ou antes de aplicar) |
| 2 | falhou, e o alvo pode ter ficado no meio (sem rollback, ou a volta falhou) |
| 3 | recusado (autenticação, confiança, validação) |
| 4 | ocupado, sem resposta, ou tempo esgotado esperando |
