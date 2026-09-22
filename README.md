# Chatwoot self-hosted

Stack de produção do Chatwoot para uma VPS: Rails + Sidekiq + Postgres (pgvector)
+ Redis + Caddy com HTTPS automático (Let's Encrypt).

> **Usa Claude Code?** Rode `/deploy-chatwoot` — a skill versionada em
> `.claude/skills/` conduz o deploy do zero, verifica cada etapa e diagnostica falhas.

**Antes de subir, preencha o [`SECRETS.md`](./SECRETS.md).** Ele lista cada
variável, como gerar os segredos e os pré-requisitos de DNS/firewall.

---

## Deploy pelo Portainer (recomendado)

1. **Stacks → Add stack → Repository**
2. Repository URL: `https://github.com/pedrourio/chatwoot-instance` · Compose path: `docker-compose.yaml`
3. Em **Environment variables**, clique em *Advanced mode* e cole o conteúdo do
   seu `.env` preenchido (use o [`.env.example`](./.env.example) como base).
4. **Deploy the stack.**

A primeira subida leva ~3–5 min: o serviço `init` roda as migrations do banco e
sai; `rails` e `sidekiq` só sobem depois que ele terminar com sucesso. O
certificado SSL é emitido pelo Caddy na primeira requisição ao domínio.

Acesse `https://SEU_DOMINIO` — a primeira tela pede a criação da conta de
administrador. **Crie essa conta imediatamente após o deploy**, antes de
divulgar a URL.

### Atualizar versão

Troque `CHATWOOT_TAG` nas variáveis do stack e use **Update the stack** com
*Re-pull image* marcado. As migrations rodam sozinhas pelo serviço `init`.

---

## Deploy pelo terminal (alternativa ao Portainer)

```bash
cp .env.example .env   # preencha conforme SECRETS.md
docker compose up -d
docker compose logs -f rails
```

---

## Já tenho um proxy na VPS (nginx/Traefik/Cloudflare Tunnel)

As portas 80/443 vão conflitar. Nesse caso:

1. Remova o serviço `caddy` (e os volumes `caddy_data` / `caddy_config`) do `docker-compose.yaml`.
2. Publique o Rails localmente adicionando ao serviço `rails`:
   ```yaml
   ports:
     - "127.0.0.1:3000:3000"
   ```
3. Aponte seu proxy para `127.0.0.1:3000` e mantenha o TLS nele.

`FRONTEND_URL` continua sendo `https://${DOMAIN}` — o Chatwoot usa esse valor
para montar links em e-mails e webhooks, então precisa ser a URL pública real.

---

## Operação

**Backup** (rode antes de qualquer atualização):

```bash
docker compose exec -T postgres pg_dump -U chatwoot chatwoot | gzip > chatwoot-$(date +%F).sql.gz
docker run --rm -v chatwoot_storage:/s -v "$PWD:/b" alpine tar czf /b/storage-$(date +%F).tar.gz -C /s .
```

O `pg_dump` cobre conversas, contatos e configurações; o `storage` cobre os
anexos enviados nas conversas. **Os dois são necessários** para uma restauração
completa.

**Restore do banco:**

```bash
gunzip -c chatwoot-AAAA-MM-DD.sql.gz | docker compose exec -T postgres psql -U chatwoot -d chatwoot
```

**Console Rails:** `docker compose exec rails bundle exec rails console`

---

## Notas

- Os dados vivem em volumes Docker nomeados (`postgres`, `redis`, `storage`).
  `docker compose down` preserva; `docker compose down -v` **apaga tudo**.
- Anexos são gravados em disco local (`ACTIVE_STORAGE_SERVICE=local`). Para
  migrar depois para S3, é trocar as variáveis de storage — o volume atual
  precisa ser copiado junto.
- `ENABLE_ACCOUNT_SIGNUP=false`: cadastro público desabilitado. Novos agentes
  entram por convite (que depende do SMTP estar configurado).
