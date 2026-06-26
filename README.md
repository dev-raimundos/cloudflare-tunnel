# Cloudflare Tunnel — Homelab Stack

> Stack Docker para expor serviços locais à internet via Cloudflare Tunnel, sem abrir portas no roteador.

---

## O que é isso

Este repositório contém a configuração para subir o **cloudflared** via Docker Compose. O cloudflared é o agente da Cloudflare que cria um túnel criptografado de saída entre o seu servidor e a rede da Cloudflare — permitindo expor serviços internos publicamente sem precisar abrir nenhuma porta no roteador ou firewall.

Todo o tráfego é roteado pelo Cloudflare Zero Trust, onde você configura os hostnames públicos e para qual serviço interno cada um aponta.

---

## Arquitetura

```
Internet
   │
   ▼
┌──────────────────────────────┐
│     Cloudflare Edge          │  ← DNS + proxy + Zero Trust
└──────────────┬───────────────┘
               │ túnel criptografado (saída)
               ▼
┌──────────────────────────────┐
│        cloudflared           │  ← agente rodando no servidor
└──────────────┬───────────────┘
               │ rede interna (home-network)
               ▼
┌──────────────────────────────┐
│     Serviços locais          │  ← NPM, Jellyfin, Home Assistant...
└──────────────────────────────┘
```

O cloudflared mantém uma conexão persistente com a Cloudflare. Quando uma requisição chega ao domínio público configurado, a Cloudflare a injeta no túnel e o cloudflared a encaminha para o serviço interno correspondente — tudo sem expor IP ou porta.

---

## Pré-requisitos

- Docker Engine 24+
- Docker Compose v2 (plugin, não o `docker-compose` legado)
- Conta na Cloudflare com um domínio configurado
- Rede Docker externa `home-network` criada no servidor
- Token de tunnel gerado no painel **Cloudflare Zero Trust**

---

## Como usar

**1. Crie a rede externa (caso ainda não exista)**

```bash
docker network create home-network
```

**2. Gere o token do tunnel no Cloudflare**

Acesse [one.dash.cloudflare.com](https://one.dash.cloudflare.com) → **Networks → Tunnels → Create a tunnel → Cloudflared → Docker**.

O painel vai exibir um comando com o token embutido. Copie apenas o valor do token (a string longa após `--token`).

**3. Crie o arquivo `.env` a partir do exemplo**

```bash
cp .env.example .env
```

Edite o `.env` com o token real:

```env
TUNNEL_TOKEN=eyJhIjoiO...
```

**4. Suba a stack**

```bash
docker compose up -d
```

**5. Verifique o status do tunnel**

```bash
docker compose logs -f
```

O tunnel aparecerá como **HEALTHY** no painel do Cloudflare Zero Trust alguns segundos após subir.

**6. Configure os hostnames públicos**

No painel do Cloudflare Zero Trust → **Networks → Tunnels → seu tunnel → Public Hostnames**, adicione as rotas:

| Subdomínio | Serviço interno |
|------------|-----------------|
| `npm.seudominio.com` | `http://nginx-proxy-manager:80` |
| `jellyfin.seudominio.com` | `http://jellyfin:8096` |

---

## Estrutura do repositório

```
.
├── compose.yaml          # definição do serviço
├── .env                  # token real (não versionar)
├── .env.example          # modelo sem valores reais (pode versionar)
├── .gitignore
└── README.md
```

---

## Variáveis de ambiente

| Variável | Descrição |
|----------|-----------|
| `TUNNEL_TOKEN` | Token gerado no Cloudflare Zero Trust ao criar o tunnel |

---

## Notas de segurança

- O arquivo `.env` **nunca** deve ser commitado no Git — ele contém o token que autentica o tunnel.
- O cloudflared não precisa de nenhuma porta exposta no host: a conexão é de saída, iniciada pelo container.
- O token é específico por tunnel. Se for comprometido, revogue-o no painel e gere um novo.
- Combine com as **Access Policies** do Cloudflare Zero Trust para proteger serviços com autenticação antes de chegarem ao serviço interno.

---

*Homelab pessoal — sem compromisso, sem frescura.*
