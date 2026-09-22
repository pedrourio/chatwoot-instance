---
name: deploy-chatwoot
description: Sobe ou atualiza a instância do Chatwoot numa VPS com Portainer. Use quando o usuário pedir para "subir o chatwoot", "fazer o deploy na VPS", "criar o stack no portainer", "atualizar a versão do chatwoot", ou quando o deploy falhou e precisa ser diagnosticado (certificado SSL, container reiniciando, migrations). Handles Portuguese or English requests.
---

# Deploy do Chatwoot no Portainer

Runbook determinístico. Execute as fases em ordem — **a fase 1 não é opcional**,
pular a checagem de DNS é a causa mais comum de deploy quebrado.

## Fase 1 — Pré-requisitos (verificar, não assumir)

Pergunte ao usuário o domínio e o IP da VPS. Então confirme:

```bash
dig +short SEU_DOMINIO          # precisa retornar o IP da VPS
```

- **Não retornou nada / IP errado** → PARE. O registro A não está propagado.
  Peça para criar/corrigir no provedor de DNS e esperar. Não prossiga: cada
  tentativa falha do Let's Encrypt conta no rate limit (5 falhas/hora por host)
  e pode travar o deploy por uma hora.
- Confirme com o usuário que **portas 80 e 443** estão livres e abertas no
  firewall da VPS. A 80 é obrigatória — o desafio ACME chega por HTTP, não HTTPS.
  Portainer normalmente usa 9443/9000, então não conflita.
- Confirme **4 GB de RAM** (2 GB é o piso apertado) e ~20 GB de disco.

Se já existe outro proxy ocupando 80/443 na VPS, siga a seção
"Já tenho um proxy" do `README.md` em vez deste caminho.

## Fase 2 — Gerar os segredos

Gere os três e **entregue ao usuário para guardar em cofre de senhas** antes de seguir:

```bash
echo "SECRET_KEY_BASE=$(openssl rand -hex 64)"
echo "POSTGRES_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)"
echo "REDIS_PASSWORD=$(openssl rand -base64 32 | tr -d '/+=' | head -c 32)"
```

Os dois últimos passam por URL de conexão, por isso o `tr` remove `/`, `+` e `=`.

Monte o bloco de variáveis a partir do `.env.example`. **Nunca escreva um `.env`
preenchido no repositório** — o destino desses valores é o formulário do Portainer.

SMTP é obrigatório na prática: sem ele não há convite de agente nem reset de
senha, ou seja, ninguém além do primeiro admin entra. Se o usuário não tiver as
credenciais ainda, diga isso explicitamente e pergunte se quer subir mesmo assim.

## Fase 3 — Criar o stack

**Portainer → Stacks → Add stack → Repository**

| Campo | Valor |
|---|---|
| Repository URL | `https://github.com/pedrourio/chatwoot-instance` |
| Compose path | `docker-compose.yaml` |
| Environment variables | *Advanced mode* → cole o bloco montado na fase 2 |

Repositório privado? O Portainer pede usuário + Personal Access Token no próprio form.

Clique em **Deploy the stack**. A primeira subida leva 3–5 min: o `init` roda as
migrations e sai, e só então `rails` e `sidekiq` sobem.

## Fase 4 — Verificar (não confie no "stack deployed")

Na ordem, porque cada passo depende do anterior:

1. **`init` saiu com código 0.** `docker compose ps -a` deve mostrar
   `init  Exited (0)`. Qualquer outro código = migrations falharam, leia
   `docker compose logs init`. Um `ERROR: relation "installation_configs" does
   not exist` no início do log é normal — acontece antes das tabelas existirem.
2. **`rails` e `sidekiq` estão `Up`** e não em loop de restart.
3. **Certificado emitido.** `docker compose logs caddy | grep -i certificate`.
   Procure a obtenção bem-sucedida; erro de ACME aqui é quase sempre DNS ou
   porta 80 fechada — volte à fase 1.
4. **App responde.** Abrir `https://SEU_DOMINIO` deve redirecionar para
   `/installation/onboarding`.
5. **Criar a conta de administrador imediatamente.** Enquanto ninguém criar, a
   tela de onboarding fica aberta para quem chegar primeiro na URL. Avise o
   usuário disso em vez de deixar para depois.

## Atualizar a versão

1. Backup primeiro, sempre (comandos no `README.md`) — dump do Postgres **e**
   tar do volume `storage`. Só o dump não restaura os anexos das conversas.
2. Troque `CHATWOOT_TAG` nas variáveis do stack.
3. **Update the stack** com *Re-pull image* marcado.
4. O `init` roda as novas migrations sozinho. Verifique a fase 4 de novo.

Consulte as releases do Chatwoot antes de pular várias versões de uma vez.

## Diagnóstico

| Sintoma | Causa provável |
|---|---|
| Caddy em loop de erro ACME | DNS não propagado, porta 80 fechada, ou rate limit já atingido (espere 1h) |
| `init` sai com código ≠ 0 | Senha do Postgres com caractere especial, ou volume de um deploy anterior com senha diferente |
| App carrega sem CSS/JS | `DOMAIN` escrito com `https://` ou barra no final |
| Convite de agente não chega | SMTP não configurado, ou domínio de envio sem SPF/DKIM |
| Login cai e volta | `SECRET_KEY_BASE` mudou entre deploys |

**Nunca rode `docker compose down -v` numa instância em uso** — a flag `-v`
apaga banco, anexos e certificados. Sem backup, é perda definitiva.
