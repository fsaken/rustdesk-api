# 🚀 Guia Completo de Deploy - RustDesk API com Coolify

## 📋 Índice
1. [Visão Geral da Arquitetura](#-visão-geral-da-arquitetura)
2. [Pré-requisitos](#-pré-requisitos)
3. [Entendendo as Portas](#-entendendo-as-portas)
4. [Configuração Passo a Passo](#-configuração-passo-a-passo)
5. [Configuração com Docker Compose](#-configuração-com-docker-compose)
6. [Integração com Coolify](#-integração-com-coolify)
7. [Configuração do Cloudflare](#-configuração-do-cloudflare)
8. [Troubleshooting](#-troubleshooting)

---

## 🏗 Visão Geral da Arquitetura

### ✅ O que pode ser proxiado (via Cloudflare/Coolify):
- **Porta 21114 (HTTP)** - Console Web Admin + API REST
  - Interface web de administração (`/_admin`)
  - API do RustDesk (`/api/*`)
  - Web Client (opcional)
  - **✅ PODE usar proxy reverso**
  - **✅ PODE usar Cloudflare**
  - **✅ RECOMENDA-SE usar HTTPS**

### ❌ O que NÃO pode ser proxiado:
- **Porta 21115 (TCP)** - Web Socket
- **Porta 21116 (TCP/UDP)** - ID Server (hbbs)
- **Porta 21117 (TCP)** - Relay Server (hbbr)
- **Porta 21118 (TCP)** - NAT type test
- **Porta 21119 (TCP)** - API Server
- **❌ Essas portas DEVEM ser expostas diretamente**
- **❌ NÃO funcionam com proxy HTTP**

---

## 📦 Pré-requisitos

### 1. Servidor RustDesk Existente
Você já deve ter um RustDesk Server rodando com:
- ✅ hbbs (ID Server) na porta 21116
- ✅ hbbr (Relay Server) na porta 21117
- ✅ Chave pública gerada (`id_ed25519.pub`)

### 2. Informações Necessárias
Antes de começar, tenha em mãos:
- ✅ IP/domínio do seu servidor RustDesk
- ✅ Conteúdo do arquivo `id_ed25519.pub`
- ✅ Domínio que será usado para a console web (ex: `rustdesk-admin.seudominio.com`)
- ✅ Acesso ao Coolify

---

## 🔌 Entendendo as Portas

| Porta | Protocolo | Serviço | Proxy? | Cloudflare? | Uso |
|-------|-----------|---------|---------|-------------|-----|
| 21114 | HTTP | Console Web/API | ✅ SIM | ✅ SIM | Interface administrativa |
| 21115 | TCP | WebSocket | ❌ NÃO | ❌ NÃO | Comunicação RustDesk |
| 21116 | TCP/UDP | ID Server | ❌ NÃO | ❌ NÃO | Registro de dispositivos |
| 21117 | TCP | Relay Server | ❌ NÃO | ❌ NÃO | Relay de conexões |
| 21118 | TCP | NAT Test | ❌ NÃO | ❌ NÃO | Teste de tipo NAT |
| 21119 | TCP | API Server | ❌ NÃO | ❌ NÃO | API interna |

---

## 🛠 Configuração Passo a Passo

### Opção 1: RustDesk Server + API Separados (RECOMENDADO para Coolify)

Esta é a melhor opção quando você já tem um RustDesk Server rodando e quer adicionar apenas a API.

#### Passo 1: Obter a Chave Pública do RustDesk Server

```bash
# Se você tem acesso ao container do RustDesk Server
docker exec -it <nome-do-container-rustdesk> cat /data/id_ed25519.pub

# Ou se você tem acesso ao volume
cat /caminho/para/volume/rustdesk/data/id_ed25519.pub
```

Copie o conteúdo completo dessa chave. Exemplo:
```
8BLLhtzUBU4WVBX6hcQZdAqrWvMgZJqJ/SX5FbIc9mc=
```

#### Passo 2: Criar Diretório de Configuração

```bash
# No seu servidor
mkdir -p ~/rustdesk-api/data
mkdir -p ~/rustdesk-api/conf
cd ~/rustdesk-api
```

#### Passo 3: Criar arquivo docker-compose.yml

Crie o arquivo `docker-compose.yml` com o conteúdo abaixo:

```yaml
version: '3.8'

services:
  rustdesk-api:
    image: lejianwen/rustdesk-api:latest
    container_name: rustdesk-api
    restart: unless-stopped
    
    environment:
      # Timezone
      - TZ=America/Sao_Paulo
      
      # Configuração do RustDesk Server (SEUS DADOS AQUI)
      - RUSTDESK_API_RUSTDESK_ID_SERVER=seu-ip-ou-dominio.com:21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=seu-ip-ou-dominio.com:21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=http://seu-ip-ou-dominio.com:21114
      
      # Chave pública do RustDesk Server
      - RUSTDESK_API_RUSTDESK_KEY=8BLLhtzUBU4WVBX6hcQZdAqrWvMgZJqJ/SX5FbIc9mc=
      
      # Idioma (pt-BR, en, es, fr, zh-CN, etc)
      - RUSTDESK_API_LANG=pt-BR
      
      # Banco de dados (sqlite por padrão)
      - RUSTDESK_API_GORM_TYPE=sqlite
      
      # Configurações da aplicação
      - RUSTDESK_API_APP_WEB_CLIENT=1  # 1=ativar webclient, 0=desativar
      - RUSTDESK_API_APP_REGISTER=false  # Permitir auto-registro
      - RUSTDESK_API_APP_SHOW_SWAGGER=0  # 1=mostrar docs Swagger
      - RUSTDESK_API_APP_TOKEN_EXPIRE=168h  # 7 dias
      
      # Segurança de login
      - RUSTDESK_API_APP_CAPTCHA_THRESHOLD=3  # Captcha após 3 erros
      - RUSTDESK_API_APP_BAN_THRESHOLD=0  # 0=não banir, >0=banir após N erros
      
      # JWT (deixe vazio se não usar o lejianwen/rustdesk-server)
      - RUSTDESK_API_JWT_KEY=
      - RUSTDESK_API_JWT_EXPIRE_DURATION=168h
      
      # Configurações de proxy reverso (IMPORTANTE)
      - RUSTDESK_API_GIN_TRUST_PROXY=  # Deixe vazio para confiar em todos
      
      # Admin
      - RUSTDESK_API_ADMIN_TITLE=RustDesk Admin Console
      
    ports:
      # APENAS a porta 21114 precisa ser exposta
      - "21114:21114"
    
    volumes:
      # Persistência do banco de dados SQLite
      - ./data:/app/data
      
      # (OPCIONAL) Se quiser customizar configurações
      # - ./conf/config.yaml:/app/conf/config.yaml:ro
      
    networks:
      - rustdesk-network
    
    healthcheck:
      test: ["CMD", "wget", "--quiet", "--tries=1", "--spider", "http://localhost:21114/api/version"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s

networks:
  rustdesk-network:
    driver: bridge
```

#### Passo 4: Ajustar as Variáveis de Ambiente

**🔴 IMPORTANTE**: Substitua os valores abaixo pelos seus:

```yaml
# Exemplo com IP público
- RUSTDESK_API_RUSTDESK_ID_SERVER=203.0.113.10:21116
- RUSTDESK_API_RUSTDESK_RELAY_SERVER=203.0.113.10:21117
- RUSTDESK_API_RUSTDESK_API_SERVER=http://203.0.113.10:21114

# OU com domínio
- RUSTDESK_API_RUSTDESK_ID_SERVER=rustdesk.seudominio.com:21116
- RUSTDESK_API_RUSTDESK_RELAY_SERVER=rustdesk.seudominio.com:21117
- RUSTDESK_API_RUSTDESK_API_SERVER=http://rustdesk.seudominio.com:21114

# Sua chave pública (obtida no Passo 1)
- RUSTDESK_API_RUSTDESK_KEY=SUA_CHAVE_AQUI
```

#### Passo 5: Subir o Container

```bash
cd ~/rustdesk-api
docker-compose up -d
```

#### Passo 6: Verificar Logs e Senha do Admin

```bash
# Ver logs
docker-compose logs -f

# Procure por algo assim:
# Admin Password Is: AbC12DeF
```

**⚠️ IMPORTANTE**: Guarde essa senha! É a senha inicial do usuário `admin`.

#### Passo 7: Testar Acesso Local

```bash
# Testar API
curl http://localhost:21114/api/version

# Deve retornar algo como:
# {"version":"v2.x.x","api_version":"v2"}
```

---

## 🔧 Integração com Coolify

### Método 1: Docker Compose no Coolify (RECOMENDADO)

#### 1. No Coolify, criar novo serviço:
1. Acesse seu Coolify
2. Vá em **Projects** → Seu projeto
3. Clique em **+ New Resource**
4. Selecione **Docker Compose**

#### 2. Configurar o serviço:

**Nome**: `rustdesk-api`

**Docker Compose** (cole o conteúdo do docker-compose.yml acima)

#### 3. Configurar Domínio e Proxy:

Na seção **Domains**:
- Adicione: `rustdesk-admin.seudominio.com`
- ✅ Marque **Enable HTTPS**
- ✅ Marque **Force HTTPS**

Na seção **Network**:
- **Port**: `21114`
- **Proxy Port**: (deixe vazio, Coolify detecta automaticamente)

#### 4. Variáveis de Ambiente Adicionais para Coolify:

```bash
# IMPORTANTE: Configure a URL da API com HTTPS quando usar proxy
RUSTDESK_API_RUSTDESK_API_SERVER=https://rustdesk-admin.seudominio.com
```

#### 5. Deploy:
- Clique em **Deploy**
- Aguarde o deploy finalizar
- Verifique os logs para pegar a senha do admin

---

### Método 2: Dockerfile Customizado no Coolify

Se preferir usar um Dockerfile ao invés do docker-compose:

#### 1. Criar um novo repositório Git com:

**Dockerfile**:
```dockerfile
FROM lejianwen/rustdesk-api:latest

# Expor apenas a porta web
EXPOSE 21114

# Healthcheck
HEALTHCHECK --interval=30s --timeout=10s --retries=3 --start-period=40s \
  CMD wget --quiet --tries=1 --spider http://localhost:21114/api/version || exit 1
```

**docker-compose.yml** (para referência local):
```yaml
# Use o mesmo conteúdo do exemplo anterior
```

#### 2. No Coolify:
1. **+ New Resource** → **Public Repository**
2. Cole a URL do seu repositório
3. Configure as variáveis de ambiente
4. Configure o domínio
5. Deploy!

---

## ☁️ Configuração do Cloudflare

### Cenário 1: Apenas Console Web via Cloudflare (RECOMENDADO)

```
Internet → Cloudflare Proxy → Coolify Proxy → Container (porta 21114)
                ↓ HTTPS ↓          ↓ HTTP ↓         ↓ HTTP ↓
```

#### Configuração DNS no Cloudflare:

| Type | Name | Content | Proxy Status | TTL |
|------|------|---------|--------------|-----|
| A | rustdesk-admin | IP_DO_SERVIDOR | ✅ Proxied | Auto |

**Configurações SSL/TLS no Cloudflare:**
- SSL/TLS encryption mode: **Full** (ou Full Strict se tiver certificado válido)
- Always Use HTTPS: **On**
- Automatic HTTPS Rewrites: **On**

**Configurações do Firewall (opcional mas recomendado):**
```
# Bloquear acesso direto à porta 21114
- Criar regra para permitir apenas tráfego do Cloudflare
- IPs do Cloudflare: https://www.cloudflare.com/ips/
```

---

### Cenário 2: Servidor RustDesk Completo com Cloudflare

**⚠️ ATENÇÃO**: Apenas a porta 21114 pode usar Cloudflare!

#### DNS Configuration:

| Type | Name | Content | Proxy Status | Porta | Uso |
|------|------|---------|--------------|-------|-----|
| A | rustdesk-admin | IP_SERVIDOR | ✅ Proxied | 21114 | Console Web |
| A | rustdesk | IP_SERVIDOR | ⚠️ DNS Only | 21116/21117 | RustDesk Server |

**Configuração nos Clientes RustDesk:**
```
ID Server: rustdesk.seudominio.com:21116
Relay Server: rustdesk.seudominio.com:21117
API Server: https://rustdesk-admin.seudominio.com
Key: SUA_CHAVE_PUBLICA
```

---

## 🔐 Segurança Pós-Deploy

### 1. Trocar Senha do Admin

```bash
# Via CLI no container
docker exec -it rustdesk-api ./apimain reset-admin-pwd NovaSenhaForte123!

# Ou acesse https://rustdesk-admin.seudominio.com/_admin/
# Faça login com admin e a senha inicial
# Vá em configurações e troque a senha
```

### 2. Criar Usuários Adicionais

1. Acesse `https://rustdesk-admin.seudominio.com/_admin/`
2. Login com `admin`
3. Vá em **Usuários** → **Criar Novo**
4. Preencha os dados
5. Defina permissões (Admin ou Usuário)

### 3. Configurar LDAP/OAuth (Opcional)

Se você tem Active Directory ou quer login com Google/GitHub:

```yaml
# No docker-compose.yml, adicione:
environment:
  # LDAP
  - RUSTDESK_API_LDAP_ENABLE=true
  - RUSTDESK_API_LDAP_URL=ldap://seu-ad-server.com:389
  - RUSTDESK_API_LDAP_BIND_DN=CN=Service Account,DC=example,DC=com
  - RUSTDESK_API_LDAP_BIND_PASSWORD=senha_service_account
  - RUSTDESK_API_LDAP_BASE_DN=DC=example,DC=com
  
  # OAuth (configure também via Web Admin)
  # GitHub, Google, etc.
```

### 4. Configurar Backup Automático

```bash
# Criar script de backup
cat > ~/backup-rustdesk-api.sh << 'EOF'
#!/bin/bash
BACKUP_DIR="/backups/rustdesk-api"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"
docker exec rustdesk-api tar czf - /app/data | \
  gzip > "$BACKUP_DIR/rustdesk-api-$DATE.tar.gz"

# Manter apenas últimos 7 backups
find "$BACKUP_DIR" -name "rustdesk-api-*.tar.gz" -mtime +7 -delete

echo "Backup concluído: $BACKUP_DIR/rustdesk-api-$DATE.tar.gz"
EOF

chmod +x ~/backup-rustdesk-api.sh

# Adicionar ao crontab (backup diário às 2am)
crontab -e
# Adicione: 0 2 * * * /root/backup-rustdesk-api.sh
```

---

## 📊 Monitoramento

### Healthcheck

```bash
# Via curl
curl -f http://localhost:21114/api/version || exit 1

# Via wget
wget --quiet --tries=1 --spider http://localhost:21114/api/version || exit 1
```

### Logs

```bash
# Ver logs em tempo real
docker-compose logs -f rustdesk-api

# Últimas 100 linhas
docker-compose logs --tail=100 rustdesk-api

# Logs de erro apenas
docker-compose logs rustdesk-api | grep -i error
```

### Métricas Importantes

Monitore via interface web (`/_admin/`):
- Total de usuários ativos
- Total de dispositivos conectados
- Logs de login
- Logs de conexão
- Logs de transferência de arquivos

---

## 🔧 Troubleshooting

### Problema 1: "Admin Password não aparece nos logs"

**Solução**:
```bash
# Resetar senha manualmente
docker exec -it rustdesk-api ./apimain reset-admin-pwd SenhaForte123!
```

### Problema 2: "502 Bad Gateway no Coolify"

**Causas comuns**:
1. Container não está rodando
   ```bash
   docker ps | grep rustdesk-api
   ```

2. Porta errada configurada no Coolify
   - Verifique se está usando porta **21114**

3. Container ainda está iniciando
   - Aguarde 30-60 segundos após o deploy

**Solução**:
```bash
# Verificar status
docker-compose ps

# Verificar logs
docker-compose logs rustdesk-api

# Testar porta localmente
curl http://localhost:21114/api/version
```

### Problema 3: "Clientes RustDesk não conectam"

**Checklist**:
- ✅ Servidor RustDesk (hbbs/hbbr) está rodando?
- ✅ Portas 21116/21117 estão abertas no firewall?
- ✅ A chave pública está correta?
- ✅ ID Server e Relay Server estão configurados corretamente?

**Teste de conexão**:
```bash
# Testar porta ID Server
nc -zv seu-servidor.com 21116

# Testar porta Relay Server
nc -zv seu-servidor.com 21117
```

### Problema 4: "ERR_TOO_MANY_REDIRECTS no Cloudflare"

**Causa**: Conflito de SSL/TLS

**Solução**:
1. No Cloudflare: SSL/TLS → Overview
2. Mude para **"Full"** (não Full Strict)
3. Aguarde 5 minutos
4. Teste novamente

### Problema 5: "Database migration error"

**Solução**:
```bash
# Parar container
docker-compose down

# Backup do banco
cp -r ./data ./data.backup

# Limpar banco (CUIDADO!)
rm -rf ./data/*

# Subir novamente
docker-compose up -d

# Verificar logs
docker-compose logs -f
```

### Problema 6: "LDAP authentication failed"

**Debug**:
```bash
# Ver logs detalhados
docker-compose logs rustdesk-api | grep -i ldap

# Testar conexão LDAP manualmente
docker exec -it rustdesk-api sh
apk add openldap-clients
ldapsearch -x -H ldap://seu-servidor:389 -D "CN=user,DC=example,DC=com" -W
```

---

## 📚 Referências e Documentação

### Links Úteis:
- **Repositório Original**: https://github.com/lejianwen/rustdesk-api
- **RustDesk Server**: https://github.com/lejianwen/rustdesk-server
- **RustDesk Oficial**: https://rustdesk.com/
- **Coolify**: https://coolify.io/

### Estrutura de Pastas:
```
~/rustdesk-api/
├── docker-compose.yml          # Configuração do Docker
├── data/                        # Dados persistentes
│   ├── rustdesk.db             # Banco SQLite
│   └── cache/                  # Cache da aplicação
└── conf/                        # (Opcional) Configurações
    └── config.yaml             # Config customizado
```

### Endpoints Importantes:
```
https://rustdesk-admin.seudominio.com/
  ├── /_admin/              # Interface administrativa
  ├── /api/                 # API REST do RustDesk
  │   ├── /api/version      # Versão da API
  │   ├── /api/login        # Login
  │   ├── /api/ab           # Address Book
  │   └── /api/peers        # Peers (dispositivos)
  ├── /webclient            # Web client v1 (opcional)
  └── /swagger/index.html   # Documentação API (se habilitado)
```

---

## 🎯 Configuração Recomendada para Produção

```yaml
version: '3.8'

services:
  rustdesk-api:
    image: lejianwen/rustdesk-api:latest
    container_name: rustdesk-api
    restart: unless-stopped
    
    environment:
      - TZ=America/Sao_Paulo
      
      # Servidor RustDesk
      - RUSTDESK_API_RUSTDESK_ID_SERVER=rustdesk.seudominio.com:21116
      - RUSTDESK_API_RUSTDESK_RELAY_SERVER=rustdesk.seudominio.com:21117
      - RUSTDESK_API_RUSTDESK_API_SERVER=https://rustdesk-admin.seudominio.com
      - RUSTDESK_API_RUSTDESK_KEY=SUA_CHAVE_AQUI
      
      # Aplicação
      - RUSTDESK_API_LANG=pt-BR
      - RUSTDESK_API_GORM_TYPE=sqlite
      - RUSTDESK_API_APP_WEB_CLIENT=1
      - RUSTDESK_API_APP_REGISTER=false  # Desabilitar auto-registro
      - RUSTDESK_API_APP_SHOW_SWAGGER=0  # Não expor Swagger
      
      # Segurança
      - RUSTDESK_API_APP_CAPTCHA_THRESHOLD=3
      - RUSTDESK_API_APP_BAN_THRESHOLD=5  # Banir após 5 tentativas
      - RUSTDESK_API_APP_TOKEN_EXPIRE=24h  # Token expira em 24h
      
      # MySQL (OPCIONAL - para ambientes grandes)
      # - RUSTDESK_API_GORM_TYPE=mysql
      # - RUSTDESK_API_MYSQL_USERNAME=rustdesk
      # - RUSTDESK_API_MYSQL_PASSWORD=senha_forte_aqui
      # - RUSTDESK_API_MYSQL_ADDR=mysql:3306
      # - RUSTDESK_API_MYSQL_DBNAME=rustdesk
      
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
      start_period: 40s
    
    # Limites de recursos (recomendado)
    deploy:
      resources:
        limits:
          cpus: '1.0'
          memory: 512M
        reservations:
          cpus: '0.5'
          memory: 256M

networks:
  rustdesk-network:
    driver: bridge
```

---

## ✅ Checklist Final

Antes de considerar o deploy concluído:

- [ ] Container está rodando (`docker ps`)
- [ ] Logs não mostram erros críticos
- [ ] Acesso web funciona (`https://rustdesk-admin.seudominio.com/_admin/`)
- [ ] Login com usuário `admin` funciona
- [ ] Senha do admin foi alterada
- [ ] SSL/TLS está ativo (cadeado verde no navegador)
- [ ] Cloudflare está protegendo o tráfego (se configurado)
- [ ] Backup está configurado
- [ ] Clientes RustDesk conseguem se conectar ao servidor
- [ ] Documentação está salva em local seguro

---

## 🆘 Suporte

Se encontrar problemas:

1. **Verifique os logs primeiro**: `docker-compose logs -f`
2. **Consulte a documentação oficial**: https://github.com/lejianwen/rustdesk-api/wiki
3. **Issues no GitHub**: https://github.com/lejianwen/rustdesk-api/issues
4. **Comunidade RustDesk**: https://rustdesk.com/community

---

**Desenvolvido com ❤️ pela comunidade RustDesk**

**Última atualização**: Novembro 2025
