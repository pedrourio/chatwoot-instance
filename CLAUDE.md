# chatwoot-instance

Stack de produção do Chatwoot self-hosted para uma VPS com Portainer.
Este repositório contém **apenas configuração de deploy** — nenhum código de aplicação.

## Arquitetura

`docker-compose.yaml` define 6 serviços:

| Serviço | Papel |
|---|---|
| `postgres` | pgvector/pg16 — o Chatwoot exige a extensão vector |
| `redis` | fila do Sidekiq e cache, com senha |
| `init` | roda `rails db:chatwoot_prepare` e sai. `rails` e `sidekiq` têm gate `service_completed_successfully` nele |
| `rails` | servidor web, porta 3000 interna |
| `sidekiq` | jobs assíncronos (e-mail, webhooks, automações) |
| `caddy` | TLS automático Let's Encrypt, proxy para `rails:3000` |

O Caddyfile é **inline** no compose (`configs:` com `content:`), não um arquivo no disco.
Isso é deliberado: com bind mount, um deploy via Web editor do Portainer criaria um
diretório vazio no lugar do arquivo e o Caddy subiria sem erro servindo nada.

Ao editar esse bloco, `$` do Caddy precisa ser escrito `$$` para o Compose não interpolar.

## Regras

- **Nunca comitar `.env`.** Já está no `.gitignore`. Segredos vivem no formulário de Environment variables do Portainer.
- **`SECRET_KEY_BASE` é imutável após o primeiro deploy.** Trocar invalida sessões e quebra tokens criptografados das integrações de canal. Não sugira regenerar.
- **A imagem é fixada** em `CHATWOOT_TAG`. Não troque para `latest`: atualização precisa ser uma decisão consciente, com backup antes.
- **DNS antes do stack.** Subir com o registro A não propagado gasta tentativas do rate limit do Let's Encrypt (5 falhas/hora por host).
- Alterou o compose? Valide com `docker compose config -q` antes de comitar.

## Deploy

Use a skill `deploy-chatwoot`. Ela cobre pré-requisitos, geração de segredos,
criação do stack no Portainer e verificação pós-deploy.

Documentos para o cliente: `SECRETS.md` (o que preencher) e `README.md` (como operar).
