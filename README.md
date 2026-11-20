# 🏗️ Documentação: ARR Stack Completo + Cloudflare Tunnel

Este documento detalha a instalação, configuração e integração de um servidor de mídia automatizado (ARR Stack) baseado no tutorial da Automation Avenue, mas com **melhorias críticas de segurança, acesso externo via Cloudflare e novos aplicativos (Seerr, Unpackerr, Watchtower)**.

Ele foi escrito com base em testes reais, documentando os erros encontrados e as soluções definitivas para cada um.

---

## 📋 Sumário
1. [Pré-requisitos e Estrutura de Pastas](#1-pré-requisitos-e-estrutura-de-pastas)
2. [O Arquivo `docker-compose.yml` Definitivo](#2-o-arquivo-docker-composeyml-definitivo)
3. [Configuração do Cloudflare Tunnel](#3-configuração-do-cloudflare-tunnel)
4. [Configuração Inicial (Passo a Passo)](#4-configuração-inicial-passo-a-passo)
5. [Segurança (Cloudflare Access - Zero Trust)](#5-segurança-cloudflare-access---zero-trust)
6. [Troubleshooting (Solução de Erros Comuns)](#6-troubleshooting-solução-de-erros-comuns)

---

## 1. Pré-requisitos e Estrutura de Pastas

O primeiro passo (e o mais importante para evitar erros de permissão `EACCES`) é criar a estrutura de pastas no host e garantir que o UID/GID do seu usuário seja o **1000**.

```bash
# 1. Instalar Docker e Docker Compose (se ainda não tiver)
# Siga as instruções oficiais do Docker para o seu sistema (Debian/Ubuntu).

# 2. Conferir seu UID/GID (deve retornar uid=1000)
id

# 3. Criar a estrutura de pastas do TRaSH Guides
sudo mkdir -p /data/{torrents/{tv,movies,music},media/{tv,movies,music}}

# 4. Criar pastas de configuração (appdata) para TODOS os serviços
sudo mkdir -p /docker/appdata/{radarr,sonarr,lidarr,bazarr,prowlarr,qbittorrent,jellyfin,jellyseerr,heimdall}

# 5. Aplicar as permissões corretas (CRUCIAL para os containers escreverem)
sudo chown -R 1000:1000 /data /docker/appdata
sudo chmod -R a=,a+rX,u+w,g+w /data

# 6. Criar a pasta do projeto e entrar nela
mkdir -p ~/arr-lab
cd ~/arr-lab
2. O Arquivo docker-compose.yml Definitivo
Crie o arquivo com nano docker-compose.yml e cole o conteúdo abaixo.

⚠️ Atenção aos comentários: Você precisará preencher as API Keys do Unpackerr e o token do Cloudflare posteriormente.

yaml
x-common-keys: &common-keys
    restart: unless-stopped
    logging:
      driver: json-file
      options:
        max-size: "10m"
        max-file: "3"
    environment:
      PUID: 1000
      PGID: 1000
      TZ: America/Sao_Paulo
    dns:
      - 1.1.1.1
      - 1.0.0.1

services:
  # --- GERENCIADORES (*ARR) ---
  radarr:
    <<: *common-keys
    container_name: radarr
    image: ghcr.io/hotio/radarr:latest
    ports: ["7878:7878"]
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/radarr:/config
      - /data:/data

  sonarr:
    <<: *common-keys
    container_name: sonarr
    image: ghcr.io/hotio/sonarr:latest
    ports: ["8989:8989"]
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/sonarr:/config
      - /data:/data

  lidarr:
    <<: *common-keys
    container_name: lidarr
    image: ghcr.io/hotio/lidarr:latest
    ports: ["8686:8686"]
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/lidarr:/config
      - /data:/data

  bazarr:
    <<: *common-keys
    container_name: bazarr
    image: ghcr.io/hotio/bazarr:latest
    ports: ["6767:6767"]
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/bazarr:/config
      - /data/media:/data/media

  prowlarr:
    <<: *common-keys
    container_name: prowlarr
    image: ghcr.io/hotio/prowlarr:latest
    ports: ["9696:9696"]
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/prowlarr:/config

  # --- DOWNLOAD E EXTRAÇÃO ---
  qbittorrent:
    <<: *common-keys
    container_name: qbittorrent
    image: ghcr.io/hotio/qbittorrent:latest
    ports:
      - 8080:8080
      - 6881:6881
      - 6881:6881/udp
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
      - WEBUI_PORT=8080
      - TORRENTING_PORT=6881
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/qbittorrent:/config
      - /data:/data

  unpackerr:
    <<: *common-keys
    container_name: unpackerr
    image: golift/unpackerr:latest
    volumes:
      - /data:/data
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
      - UN_LOG_FILE=/data/unpackerr.log
      - UN_SONARR_0_URL=http://sonarr:8989
      - UN_SONARR_0_API_KEY=PREENCHER_CHAVE_AQUI
      - UN_RADARR_0_URL=http://radarr:7878
      - UN_RADARR_0_API_KEY=PREENCHER_CHAVE_AQUI

  # --- MÍDIA E FRONTEND ---
  jellyfin:
    <<: *common-keys
    container_name: jellyfin
    image: ghcr.io/hotio/jellyfin:latest
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /docker/appdata/jellyfin:/config
      - /data/media:/data/media:ro
    ports: ["8096:8096"]
    # devices:
    #   - /dev/dri:/dev/dri  # Descomente para aceleração de hardware

  jellyseerr: # Seerr (antigo Jellyseerr)
    container_name: jellyseerr
    image: ghcr.io/seerr-team/seerr:latest
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
    ports: ["5055:5055"]
    volumes:
      - /docker/appdata/jellyseerr:/app/config
    environment:
      - TZ=America/Sao_Paulo
      - LOG_LEVEL=info
    # Nota: O Seerr novo roda rootless (UID 1000 interno), não aceita PUID/PGID.

  heimdall:
    <<: *common-keys
    container_name: heimdall
    image: lscr.io/linuxserver/heimdall:latest
    ports:
      - 8085:80
      - 8443:443
    volumes:
      - /docker/appdata/heimdall:/config

  # --- UTILITÁRIOS ---
  flaresolverr:
    <<: *common-keys
    container_name: flaresolverr
    image: ghcr.io/flaresolverr/flaresolverr:latest
    ports: ["8191:8191"]
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Sao_Paulo
      - LOG_LEVEL=info

  watchtower:
    image: containrrr/watchtower:latest
    container_name: watchtower
    restart: unless-stopped
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
    environment:
      - TZ=America/Sao_Paulo
      - WATCHTOWER_SCHEDULE=0 0 17 * * MON # Roda toda segunda às 17h
      - WATCHTOWER_CLEANUP=true
      - WATCHTOWER_TIMEOUT=90s

  # --- TÚNEL CLOUDFLARE ---
  cloudflared:
    image: cloudflare/cloudflared:latest
    container_name: cloudflared-lab
    restart: always
    command: tunnel --no-autoupdate run --token SUBSTITUA_PELO_TOKEN_DO_SEU_TUNEL
    logging:
      driver: json-file
      options: { max-size: "10m", max-file: "3" }
    dns:
      - 1.1.1.1
      - 1.0.0.1
    depends_on:
      - jellyfin
      - radarr
      - sonarr
      - jellyseerr

networks:
  default:
    name: arr_network
3. Configuração do Cloudflare Tunnel
O Cloudflare Tunnel permite acessar seus serviços de fora de casa sem abrir portas no roteador.

No painel Cloudflare, vá em Zero Trust > Networks > Tunnels e crie um túnel.

Copie o token gerado e cole no docker-compose.yml (no comando do cloudflared).

Na aba Public Hostnames, adicione as rotas para cada serviço.

⚠️ Regra de Ouro: O campo "URL" deve ser nome_do_servico:porta_interna, sem http://. Exemplo: jellyfin:8096.

Se der erro 502 Bad Gateway, é porque você colocou http:// no campo URL ou a porta está errada.

Subdomain	Domain	Service Type	URL
jellyfin	seudominio.com	HTTP	jellyfin:8096
radarr	seudominio.com	HTTP	radarr:7878
sonarr	seudominio.com	HTTP	sonarr:8989
prowlarr	seudominio.com	HTTP	prowlarr:9696
qbit	seudominio.com	HTTP	qbittorrent:8080
seerr	seudominio.com	HTTP	jellyseerr:5055
heimdall	seudominio.com	HTTP	heimdall:80
4. Configuração Inicial (Passo a Passo)
Suba os containers com docker compose up -d. Acesse via IP local (http://192.168.x.x:porta) e siga a ordem abaixo:

4.1. qBittorrent (O Coração dos Downloads)
Senha Temporária: Rode docker compose logs qbittorrent. Procure por The WebUI administrator password is....

Acesse http://IP:8080 e faça login.

Vá em Tools > Options > WebUI e mude a senha (salve rolando a página).

Vá em Categories (menu esquerdo), clique com o botão direito e crie:

movies (Save Path: movies)

tv (Save Path: tv)

music (Save Path: music)

Vá em Tools > Options > Downloads:

Marque Default Torrent Management Mode: Automatic.

Marque as opções de relocação ao mudar categoria.

Marque Use Subcategories e Use Category paths in Manual Mode.

Default Save Path: /data/torrents. Salve.

4.2. Prowlarr (Gerenciador de Indexers)
Acesse http://IP:9696. Vá em Settings > Download Clients.

Adicione o qBittorrent: Host qbittorrent, Porta 8080, desmarque SSL, use o usuário e senha criados.

4.3. FlareSolverr (O Bypass da Cloudflare nos Indexers)
Muitos indexers são bloqueados pela Cloudflare. O FlareSolverr resolve isso.

No Prowlarr, vá em Settings > Indexers.

Clique no + (Indexer Proxies) e escolha FlareSolverr.

Preencha:

Name: FlareSolverr

Host: http://flaresolverr:8191

Tags: cloudflare (Esta tag é vital).

Salve. Como usar: Vá no seu indexer que está bloqueando. Na aba de configuração dele, adicione a tag cloudflare no campo "Tags".

4.4. Radarr, Sonarr e Lidarr (Os Gerenciadores)
Faça o mesmo processo para os três (Portas 7878, 8989, 8686):

Root Folder: Vá em Settings > Media Management > Add Root Folder.

Radarr: /data/media/movies

Sonarr: /data/media/tv

Lidarr: /data/media/music

Hardlinks: Ainda em Media Management, clique em Show Advanced e marque Use Hardlinks instead of Copy (economiza espaço em disco).

Download Client: Vá em Settings > Download Clients, adicione o qBittorrent (Host qbittorrent, Porta 8080). A categoria deve ser movies, tv ou music.

API Keys: Vá em Settings > General e copie a API Key.

4.5. Sincronizar o Prowlarr com os *ARR
No Prowlarr, vá em Settings > Apps.

Clique no + e adicione o Radarr. Cole a API Key.

Nas URLs dos servidores, use os nomes dos containers: http://radarr:7878, http://sonarr:8989, http://lidarr:8686.

Clique em Test (tem que ficar verde) e salve.

4.6. Jellyfin (O Servidor de Streaming)
Acesse http://IP:8096 e crie o administrador.

Adicione as bibliotecas apontando para /data/media/movies, /data/media/tv e /data/media/music.

Ajustes para Cloudflare: Vá em Dashboard > Networking:

Published server URL: https://jellyfin.seudominio.com (seu domínio).

Known Proxies: Coloque a sub-rede do Docker. Descubra rodando docker network inspect arr_network e procurando o campo Subnet (geralmente 172.18.0.0/16).

Salve e reinicie: docker compose restart jellyfin.

4.7. Seerr (Jellyseerr)
Acesse o Seerr via IP (5055) ou domínio.

⚠️ Se der erro EACCES: permission denied: Pare o container, rode sudo chown -R 1000:1000 /docker/appdata/jellyseerr, remova o container antigo (docker compose rm -sf jellyseerr) e suba de novo.

Siga o Wizard:

Jellyfin URL: Apenas jellyfin (sem http://).

Port: 8096

Use SSL: Desmarcado.

Preencha e-mail, usuário e senha do Jellyfin.

Configuração do Jellyfin: Coloque a URL externa https://jellyfin.seudominio.com e clique em Iniciar Digitalização.

Configurar Serviços: Adicione o Radarr (Host radarr, porta 7878) e Sonarr (Host sonarr, porta 8989) com suas respectivas API Keys.

4.8. Unpackerr (Extrator Automático)
Após configurar Radarr e Sonarr, copie as API Keys deles.

Abra o docker-compose.yml, vá no bloco do unpackerr e substitua PREENCHER_CHAVE_AQUI pelas chaves reais.

Recarregue apenas ele: docker compose up -d unpackerr.

4.9. Heimdall e Watchtower
O Heimdall (porta 8085) já está pronto para você criar os atalhos.

O Watchtower já está rodando. Ele atualizará as imagens automaticamente toda segunda-feira às 17h.

5. Segurança (Cloudflare Access - Zero Trust)
Para proteger o acesso externo ao qBittorrent, Radarr, etc., adicione uma camada de login extra:

Vá em Zero Trust > Settings > Authentication e ative o One-time PIN.

Vá em Access > Applications e crie uma aplicação.

Nome: Painéis ARR. Domínio: *.seudominio.com (ou específico).

Crie uma Policy de Allow para o seu e-mail.

Salve. Agora, antes de chegar na tela de login, a Cloudflare pede um código no seu e-mail.

6. Troubleshooting (Solução de Erros Comuns)
Erro 502 Bad Gateway no Cloudflare: Você colocou http:// no campo URL da rota no painel, ou a porta está errada.

Erro EACCES: permission denied no Seerr/Jellyseerr: A imagem nova do Seerr roda rootless. Rode sudo chown -R 1000:1000 /docker/appdata/jellyseerr e reinicie.

*Os ARR não movem o arquivo (Hardlink não funciona): Verifique se as pastas /data/torrents e /data/media estão no mesmo sistema de arquivos e se o chown 1000:1000 foi aplicado.

qBittorrent lento / atrás de NAT: Lembre-se de redirecionar a porta 6881 (TCP e UDP) no seu roteador para o IP do Debian.

Jellyfin não abre pelo domínio: Verifique se a URL na Cloudflare está como jellyfin:8096 (sem http://).
