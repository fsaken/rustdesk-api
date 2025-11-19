# 🚀 RustDesk API com Coolify - Guia Rápido

Guia simplificado para deploy da API do RustDesk (Web Console) integrado com Coolify e Cloudflare.

## ⚡ Quick Start (Docker Compose)

1.  **Obtenha a Chave Pública** do seu RustDesk Server (`id_ed25519.pub`).
2.  **Crie o arquivo `docker-compose.yml`**:

```yaml
version: '3.8'

services:
  rustdesk-api:
    image: lejianwen/rustdesk-api:latest
    container_name: rustdesk-api
    restart: unless-stopped
    environment:
      - TZ=America/Sao_Paulo
      # Configuração do RustDesk Server
      - RUSTDESK_API_RUSTDESK_ID_SERVER=rustdesk.seudominio.com:21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=rustdesk.seudominio.com:21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=https://rustdesk-admin.seudominio.com
      - RUSTDESK_API_RUSTDESK_KEY=SUA_CHAVE_PUBLICA_AQUI
      # Aplicação
      - RUSTDESK_API_LANG=pt-BR
      - RUSTDESK_API_GORM_TYPE=sqlite
      - RUSTDESK_API_APP_WEB_CLIENT=1
      - RUSTDESK_API_APP_REGISTER=false
      # Segurança
      - RUSTDESK_API_APP_CAPTCHA_THRESHOLD=3
      - RUSTDESK_API_APP_BAN_THRESHOLD=5
      - RUSTDESK_API_APP_TOKEN_EXPIRE=24h
    ports:
      - "21114:21114"
    volumes:
      - ./data:/app/data
    networks:
      - rustdesk-network
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:21114/api/version"]
      interval: 30s
      timeout: 10s
      retries: 3

networks:
  rustdesk-network:
    driver: bridge
```

3.  **Inicie o serviço**: `docker-compose up -d`
4.  **Pegue a senha inicial**: `docker-compose logs rustdesk-api | grep "Admin Password"`

---

## 🔌 Portas e Arquitetura

Apenas a **porta 21114 (HTTP)** deve ser passada pelo proxy (Coolify/Cloudflare). As outras portas (21115-21119) são para conexão direta dos clientes RustDesk.

| Porta | Serviço | Proxy? |
|-------|---------|--------|
| **21114** | **Web Console / API** | ✅ **SIM** (HTTP/HTTPS) |
| 21116 | ID Server | ❌ NÃO (TCP/UDP) |
| 21117 | Relay Server | ❌ NÃO (TCP) |

---

## 🔧 Integração com Coolify

1.  Crie um novo serviço **Docker Compose** no Coolify.
2.  Cole o conteúdo do `docker-compose.yml` acima.
3.  Nas configurações de **Domains**, adicione seu domínio (ex: `rustdesk-admin.seudominio.com`) e ative **HTTPS**.
4.  Em **Environment Variables**, certifique-se de que `RUSTDESK_API_RUSTDESK_API_SERVER` comece com `https://`.

---

## ☁️ Cloudflare (Opcional)

Se usar Cloudflare para o domínio da console web:
1.  Crie um registro **A** para `rustdesk-admin` apontando para o IP do Coolify.
2.  **Proxy Status**: ✅ Proxied (Laranja).
3.  **SSL/TLS**: Use modo **Full**.

> **Nota**: Não ative o proxy (nuvem laranja) para os subdomínios do servidor RustDesk (hbbs/hbbr) nas portas 21116/21117.

---

## 🆘 Troubleshooting Comum

-   **Senha não aparece**: Resete manualmente: `docker exec -it rustdesk-api ./apimain reset-admin-pwd NovaSenha!`
-   **Erro 502**: Verifique se o container está rodando e se a porta interna no Coolify está `21114`.
-   **Loop de Redirecionamento**: No Cloudflare, mude SSL para **Full**.

---

**Links Úteis**: [Repositório Oficial](https://github.com/lejianwen/rustdesk-api) | [RustDesk](https://rustdesk.com/)
