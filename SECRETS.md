# Chatwoot — variáveis que você precisa preencher

Este documento lista **tudo** que precisa ser informado para a instância subir.
Nenhum desses valores está no repositório, e nenhum deve ser comitado.

No Portainer, esses valores vão no bloco **Environment variables** do Stack
(aba *Advanced mode* aceita colar o conteúdo do `.env` de uma vez).

---

## 1. Antes de começar (pré-requisitos na VPS)

| Item | O que fazer |
|---|---|
| DNS | Criar um registro **A** do seu domínio (ex.: `chat.suaempresa.com.br`) apontando para o **IP da VPS**. Isso precisa estar propagado ANTES de subir o stack, senão o certificado SSL falha. |
| Portas 80 e 443 | Livres na VPS e liberadas no firewall. Se já existir outro nginx/Traefik ocupando essas portas, veja a seção "Já tenho um proxy" no `README.md`. |
| Disco | Mínimo 20 GB; 4 GB de RAM é o piso confortável (Rails + Sidekiq + Postgres). |

---

## 2. Segredos que VOCÊ gera (3 valores)

Rode na VPS (ou em qualquer máquina com Docker / openssl) e guarde a saída:

```bash
# SECRET_KEY_BASE — chave de criptografia de sessões do Rails
openssl rand -hex 64

# POSTGRES_PASSWORD — senha do banco
openssl rand -base64 32 | tr -d '/+=' | head -c 32; echo

# REDIS_PASSWORD — senha do Redis
openssl rand -base64 32 | tr -d '/+=' | head -c 32; echo
```

| Variável | Regra | Atenção |
|---|---|---|
| `SECRET_KEY_BASE` | 128 caracteres hex | **Nunca troque depois de subir.** Trocar invalida todas as sessões e quebra dados criptografados (tokens de integrações de canais). |
| `POSTGRES_PASSWORD` | sem `@`, `:`, `/` ou `%` | É gravada no volume do Postgres na **primeira** subida. Mudar depois exige alterar a senha dentro do banco também. |
| `REDIS_PASSWORD` | sem `@`, `:`, `/` ou `%` | Pode ser trocada; basta recriar o stack. |

---

## 3. Dados que VOCÊ informa (domínio)

| Variável | Exemplo | O que é |
|---|---|---|
| `DOMAIN` | `chat.suaempresa.com.br` | Domínio onde o Chatwoot vai responder. **Sem** `https://` e **sem** barra no final. |
| `ACME_EMAIL` | `ti@suaempresa.com.br` | E-mail usado no cadastro Let's Encrypt (recebe aviso se o certificado falhar em renovar). |

---

## 4. Servidor de e-mail (SMTP) — obrigatório

Sem SMTP o Chatwoot sobe, mas **não envia convite de agente nem reset de senha** —
na prática você não consegue adicionar ninguém no time.

Pegue esses dados com quem administra o e-mail da empresa, ou crie uma conta em
um serviço transacional (Amazon SES, Resend, Brevo, Mailgun, SendGrid).

| Variável | Exemplo | Observação |
|---|---|---|
| `SMTP_ADDRESS` | `email-smtp.us-east-1.amazonaws.com` | Host do servidor SMTP. |
| `SMTP_PORT` | `587` | Use `587` (STARTTLS). A porta `465` exige config diferente. |
| `SMTP_DOMAIN` | `suaempresa.com.br` | Seu domínio de envio. |
| `SMTP_USERNAME` | — | Usuário/API key do serviço. |
| `SMTP_PASSWORD` | — | Senha/secret do serviço. |
| `MAILER_SENDER_EMAIL` | `Suporte <suporte@suaempresa.com.br>` | Remetente exibido. O domínio precisa estar verificado (SPF/DKIM) no provedor, senão vira spam. |

> **Gmail/Google Workspace:** só funciona com *App Password* e 2FA ativo na conta;
> host `smtp.gmail.com`, porta `587`. Não use a senha normal da conta.

---

## 5. Opcionais (têm valor padrão, mexa só se quiser)

| Variável | Padrão | Para quê |
|---|---|---|
| `CHATWOOT_TAG` | `v4.18.0-ce` | Versão da imagem. Fixada de propósito — atualizar é trocar aqui e redeployar. |
| `DEFAULT_LOCALE` | `pt_BR` | Idioma padrão da interface. |
| `ENABLE_ACCOUNT_SIGNUP` | `false` | `false` = ninguém cria conta sozinho pela URL pública. Deixe assim. |
| `RAILS_MAX_THREADS` | `5` | Suba para `10` só se a VPS tiver 8 GB+ de RAM. |

---

## 6. Checklist de entrega

- [ ] Registro A do domínio apontando para o IP da VPS (propagado)
- [ ] Portas 80 e 443 liberadas no firewall
- [ ] `SECRET_KEY_BASE`, `POSTGRES_PASSWORD`, `REDIS_PASSWORD` gerados e guardados em cofre de senhas
- [ ] `DOMAIN` e `ACME_EMAIL` definidos
- [ ] Credenciais SMTP em mãos e domínio de envio verificado (SPF/DKIM)
- [ ] Stack criado no Portainer com todas as variáveis acima preenchidas
