🏗️ Manual Completo de Instalação: ARR Stack + Cloudflare + Extras
Este guia documenta a instalação do zero do seu servidor de mídia. Ele une o tutorial original da Automation Avenue com as modificações críticas que fizemos para resolver erros de permissão, adicionar a Cloudflare, o Seerr, Unpackerr e Watchtower.

🛠️ Parte 1: Preparação do Ambiente e Estrutura de Pastas
O primeiro passo é garantir que o Docker e as permissões de pastas estejam corretos. Pular esta etapa causa o erro clássico de "permission denied" nos containers.

Ele contém a preparação do ambiente, os comandos exatos para subir, validar, monitorar e recriar containers, além de todas as configurações detalhadas

🏗️ Documentação Completa: ARR Stack + Cloudflare Tunnel + Extras

Este documento detalha a instalação, configuração e integração de um servidor de mídia automatizado (ARR Stack) baseado no tutorial da Automation Avenue, mas com melhorias críticas de segurança, acesso externo via Cloudflare, novos aplicativos (Seerr, Unpackerr, Watchtower, FlareSolverr) e comandos de gerenciamento do Docker.

Ele foi escrito com base em testes reais, documentando os erros encontrados e as soluções definitivas.

📋 Sumário

Instalação do Docker e Preparação de Pastas

O Arquivo docker-compose.yml e a Primeira Execução

Gerenciamento Básico dos Containers

Configuração do Cloudflare Tunnel

Configuração Inicial (Passo a Passo)

Segurança (Cloudflare Access - Zero Trust)

Troubleshooting (Solução de Erros Comuns)

1. Instalação do Docker e Preparação de Pastas

O primeiro passo (e o mais importante para evitar erros de permissão EACCES) é preparar o sistema e a estrutura de pastas.

1.1. Instalar Docker e Docker Compose

(Se ainda não tiver instalado. Baseado no tutorial original)

# Adicionar a chave GPG oficial do Docker:
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Adicionar o repositório:
sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl status docker
sudo docker run hello-world
docker compose version


1.2. Criar a Estrutura de Pastas (TRaSH Guides)

⚠️ CRUCIAL: O Docker rodará como usuário 1000 (PUID/PGID no YAML). Se as pastas não tiverem esse dono, os containers não conseguirão gravar.

# 1. Conferir seu UID/GID (deve retornar uid=1000)
id

# 2. Criar a estrutura de pastas de mídia
sudo mkdir -p /data/{torrents/{tv,movies,music},media/{tv,movies,music}}

# 3. Criar pastas de configuração (appdata) para TODOS os serviços
sudo mkdir -p /docker/appdata/{radarr,sonarr,lidarr,bazarr,prowlarr,qbittorrent,jellyfin,jellyseerr,heimdall}

# 4. Aplicar permissões corretas
sudo chown -R 1000:1000 /data /docker/appdata
sudo chmod -R a=,a+rX,u+w,g+w /data

# 5. Conferir
tree /data
ls -ln /data /docker/appdata

# 6. Criar a pasta do projeto e entrar nela
mkdir -p ~/arr-lab
cd ~/arr-lab


2. O Arquivo docker-compose.yml e a Primeira Execução

2.1. Criar o arquivo

Crie o arquivo com nano docker-compose.yml e cole o conteúdo abaixo.

⚠️ Atenção: Antes de rodar, preencha as chaves do unpackerr (se já as tiver) e o token do cloudflared.

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


2.2. Validar e Subir os Containers

Nunca rode o up sem validar primeiro.

# 1. Validar se o YAML está sintaticamente correto
docker compose config

# 2. Subir todos os containers em background (-d)
docker compose up -d

# 3. Verificar se todos subiram corretamente (Status deve ser "Up")
docker compose ps


3. Gerenciamento Básico dos Containers

Comandos essenciais para o dia a dia no laboratório:

# Ver os logs de um container específico em tempo real
docker compose logs -f --tail=50 jellyfin

# Parar todos os containers (mantém os dados)
docker compose stop

# Iniciar os containers novamente
docker compose start

# Parar e remover os containers (NÃO apaga os dados, pois estão no host)
docker compose down

# Atualizar as imagens para a versão mais recente e recriar os containers
docker compose pull
docker compose up -d --remove-orphans


4. Configuração do Cloudflare Tunnel

O Cloudflare Tunnel permite acessar seus serviços de fora de casa sem abrir portas no roteador.

No painel Cloudflare, vá em Zero Trust > Networks > Tunnels e crie um túnel.

Copie o token gerado e cole no docker-compose.yml (no comando do cloudflared).

Na aba Public Hostnames, adicione as rotas para cada serviço.

⚠️ Regra de Ouro: O campo "URL" deve ser nome_do_servico:porta_interna, sem http://. Exemplo: jellyfin:8096.

Se der erro 502 Bad Gateway, é porque você colocou http:// no campo URL ou a porta está errada.

Subdomain

Domain

Service Type

URL

jellyfin

seudominio.com

HTTP

jellyfin:8096

radarr

seudominio.com

HTTP

radarr:7878

sonarr

seudominio.com

HTTP

sonarr:8989

prowlarr

seudominio.com

HTTP

prowlarr:9696

qbit

seudominio.com

HTTP

qbittorrent:8080

seerr

seudominio.com

HTTP

jellyseerr:5055

heimdall

seudominio.com

HTTP

heimdall:80

5. Configuração Inicial (Passo a Passo)

Acesse via IP local (http://192.168.x.x:porta) ou pelos domínios já configurados e siga a ordem abaixo:

5.1. qBittorrent (O Coração dos Downloads)

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

5.2. Prowlarr (Gerenciador de Indexers)

Acesse http://IP:9696. Vá em Settings > Download Clients.

Adicione o qBittorrent: Host qbittorrent, Porta 8080, desmarque SSL, use o usuário e senha criados.

5.3. FlareSolverr (O Bypass da Cloudflare nos Indexers)

No Prowlarr, vá em Settings > Indexers.

Clique no + (Indexer Proxies) e escolha FlareSolverr.

Preencha:

Name: FlareSolverr

Host: http://flaresolverr:8191

Tags: cloudflare (Esta tag é vital).

Salve. Como usar: Vá no seu indexer que está bloqueando. Na aba de configuração dele, adicione a tag cloudflare no campo "Tags".

5.4. Radarr, Sonarr e Lidarr (Os Gerenciadores)

Faça o mesmo processo para os três (Portas 7878, 8989, 8686):

Root Folder: Vá em Settings > Media Management > Add Root Folder.

Radarr: /data/media/movies

Sonarr: /data/media/tv

Lidarr: /data/media/music

Hardlinks: Ainda em Media Management, clique em Show Advanced e marque Use Hardlinks instead of Copy.

Download Client: Vá em Settings > Download Clients, adicione o qBittorrent (Host qbittorrent, Porta 8080). A categoria deve ser movies, tv ou music.

API Keys: Vá em Settings > General e copie a API Key.

5.5. Sincronizar o Prowlarr com os *ARR

No Prowlarr, vá em Settings > Apps.

Clique no + e adicione o Radarr. Cole a API Key.

Nas URLs dos servidores, use os nomes dos containers: http://radarr:7878, http://sonarr:8989, http://lidarr:8686.

Clique em Test (tem que ficar verde) e salve.

5.6. Jellyfin (O Servidor de Streaming)

Acesse http://IP:8096 e crie o administrador.

Adicione as bibliotecas apontando para /data/media/movies, /data/media/tv e /data/media/music.

Ajustes para Cloudflare: Vá em Dashboard > Networking:

Published server URL: https://jellyfin.seudominio.com (seu domínio).

Known Proxies: Coloque a sub-rede do Docker. Descubra rodando docker network inspect arr_network e procurando o campo Subnet (geralmente 172.18.0.0/16).

Salve e reinicie: docker compose restart jellyfin.

5.7. Seerr (Jellyseerr)

Acesse o Seerr via IP (5055) ou domínio.

⚠️ Se der erro EACCES: permission denied: Pare o container, rode sudo chown -R 1000:1000 /docker/appdata/jellyseerr, remova o container antigo (docker compose rm -sf jellyseerr) e suba de novo.

Siga o Wizard:

Jellyfin URL: Apenas jellyfin (sem http://).

Port: 8096

Use SSL: Desmarcado.

Preencha e-mail, usuário e senha do Jellyfin.

Configuração do Jellyfin: Coloque a URL externa https://jellyfin.seudominio.com e clique em Iniciar Digitalização.

Configurar Serviços: Adicione o Radarr (Host radarr, porta 7878) e Sonarr (Host sonarr, porta 8989) com suas respectivas API Keys.

5.8. Unpackerr (Extrator Automático)

Após configurar Radarr e Sonarr, copie as API Keys deles.

Abra o docker-compose.yml, vá no bloco do unpackerr e substitua PREENCHER_CHAVE_AQUI pelas chaves reais.

Recarregue apenas ele: docker compose up -d unpackerr.

5.9. Heimdall e Watchtower

O Heimdall (porta 8085) já está pronto para você criar os atalhos.

O Watchtower já está rodando. Ele atualizará as imagens automaticamente toda segunda-feira às 17h.

6. Segurança (Cloudflare Access - Zero Trust)

Para proteger o acesso externo ao qBittorrent, Radarr, etc., adicione uma camada de login extra:

Vá em Zero Trust > Settings > Authentication e ative o One-time PIN.

Vá em Access > Applications e crie uma aplicação.

Nome: Painéis ARR. Domínio: *.seudominio.com (ou específico).

Crie uma Policy de Allow para o seu e-mail.

Salve. Agora, antes de chegar na tela de login, a Cloudflare pede um código no seu e-mail.

7. Troubleshooting (Solução de Erros Comuns)

Erro 502 Bad Gateway no Cloudflare: Você colocou http:// no campo URL da rota no painel, ou a porta está errada.

Erro EACCES: permission denied no Seerr/Jellyseerr: A imagem nova do Seerr roda rootless. Rode sudo chown -R 1000:1000 /docker/appdata/jellyseerr e reinicie.

Os ARR não movem o arquivo (Hardlink não funciona): Verifique se as pastas /data/torrents e /data/media estão no mesmo sistema de arquivos e se o chown 1000:1000 foi aplicado.

qBittorrent lento / atrás de NAT: Lembre-se de redirecionar a porta 6881 (TCP e UDP) no seu roteador para o IP do Debian.

Jellyfin não abre pelo domínio: Verifique se a URL na Cloudflare está como jellyfin:8096 (sem http://).
