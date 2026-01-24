# LAB 1 — Dockerfile: Alpine + Nginx + Hello World

## Objetivo
Criar **uma imagem própria** a partir do `alpine`, instalar o **Nginx**, adicionar uma página **Hello World** e executar o container expondo a aplicação no host.

Ao final, você deve ter:
- Um diretório com `Dockerfile`, `index.html` e `.dockerignore`
- Uma imagem criada por você (ex.: `lab1-nginx:1.0`)
- Um container executando com **port mapping** (host `8080` → container `80`)
- Capacidade de inspecionar: `ps`, `logs`, `exec`, `inspect`

**Tempo sugerido:** 40–60 min.

---

## Pré-requisitos (na sua VM Debian)
1) Docker instalado e funcionando:
```bash
docker version
docker info
```

2) Se aparecer **permission denied** ao rodar `docker ...`:
- Use `sudo docker ...` **ou**
- Adicione seu usuário ao grupo docker e reabra a sessão:
```bash
sudo usermod -aG docker $USER
newgrp docker
```

---

## Checkpoint A — Criar o diretório do lab
Crie uma pasta de trabalho e entre nela:

```bash
mkdir -p ~/labs/modulo4/lab1-nginx
cd ~/labs/modulo4/lab1-nginx
```

**Validação:** `pwd` deve terminar com `.../lab1-nginx`.

---

## Checkpoint B — Criar o `index.html`
Crie a página HTML (use `vim`):

```bash
vim index.html
```

Cole o conteúdo abaixo:

```html
<!doctype html>
<html lang="pt-br">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1" />
  <title>LAB 1 - Dockerfile</title>
</head>
<body>
  <h1>Hello World (Docker + Nginx)</h1>
  <p>Imagem construída a partir do Alpine, com Nginx instalado.</p>
  <p><strong>Aluno:</strong> SEU_NOME_AQUI</p>
</body>
</html>
```

**Validação:**
```bash
ls -lah index.html
```

---

## Checkpoint C — Criar o `default.conf` e o `Dockerfile`

> **Por que isso existe?** No pacote `nginx` do Alpine, o `default.conf` padrão pode vir configurado para **retornar 404 para tudo**.  
> Aqui nós substituímos esse arquivo por uma configuração simples que **serve o `index.html`**.

### C1) Criar o `default.conf`
Crie o arquivo de configuração do Nginx (use `vim`):

```bash
vim default.conf
```

Cole este conteúdo:

```nginx
server {
  listen 80 default_server;
  server_name _;

  root /var/lib/nginx/html;
  index index.html;

  location / {
    try_files $uri $uri/ =404;
  }
}
```

**Validação:**
```bash
ls -lah default.conf
```

### C2) Criar o `Dockerfile`
Crie o arquivo Dockerfile:

```bash
vim Dockerfile
```

Cole este conteúdo:

```dockerfile
FROM alpine:3.20

LABEL maintainer="jvnetobr@fedoraproject.org" 

# Atualiza pacotes
RUN apk update

# Instala Nginx e prepara diretório de runtime
RUN apk add --no-cache nginx
RUN mkdir -p /run/nginx

# Limpa cache apk
RUN rm -rf /var/cache/apk/*

# Copia sua página para o document root padrão do pacote nginx no Alpine
COPY index.html /var/lib/nginx/html/index.html
COPY default.conf /etc/nginx/http.d/default.conf

EXPOSE 80

# Mantém o processo em foreground (necessário para o container “ficar vivo”)
CMD ["nginx", "-g", "daemon off;"]
```

**Pontos para observar (didática):**
- `FROM` define a base da imagem.
- `RUN` cria uma camada que instala pacotes.
- `COPY` “embarca” seus arquivos dentro da imagem.
- `CMD` é o processo principal do container.


---


---

## Checkpoint D — Criar o `.dockerignore`
Crie o arquivo:

```bash
vim .dockerignore
```

Conteúdo recomendado:

```gitignore
.git
.gitignore
*.log
*.tmp
node_modules
.DS_Store
```

**Por quê:** evita mandar “lixo” para o contexto de build (build mais rápido e imagem mais limpa).

---

## Checkpoint E — Build da imagem
No mesmo diretório, execute:

```bash
docker build -t lab1-nginx:1.0 .
```

**Validação:**
```bash
docker images | grep lab1-nginx
```

---

## Checkpoint F — Executar o container (port mapping)
Suba o container:

```bash
docker run -d --name lab1-web -p 8080:80 lab1-nginx:1.0
```

**Validação (host):**
```bash
curl -i http://localhost:8080 | head -n 20
```

Se estiver em ambiente gráfico, abra o navegador em `http://localhost:8080`.

---

## Checkpoint G — Inspeção do container (dia a dia)
1) Ver o container rodando:
```bash
docker ps
```

2) Ver logs:
```bash
docker logs --tail 50 lab1-web
```

3) Entrar no container e checar arquivos:
```bash
docker exec -it lab1-web sh
ls -lah /var/lib/nginx/html
ps
exit
```

4) Ver detalhes (porta, IP, mounts):
```bash
docker inspect lab1-web | less
```

---

## Exercício rápido — Cache de build
1) Edite o HTML (mude o texto):
```bash
vim index.html
```

2) Faça novo build com outra tag:
```bash
docker build -t lab1-nginx:1.1 .
```

**O que observar:** se você não mudou o `Dockerfile`, a etapa `RUN apk add ...` tende a ser reaproveitada (cache), e a mudança mais “cara” não se repete.

---

## Evidências (para entrega)
Tire prints (ou cole o output) de:
1) `docker images | grep lab1-nginx`
2) `docker ps` mostrando `lab1-web`
3) `curl -i http://localhost:8080` (ou print do navegador)

Sugestão de pasta:
- `evidencias/` com `01-images.png`, `02-ps.png`, `03-curl.png`

---

## Troubleshooting rápido
### Porta já em uso (ex.: 8080)
Erro típico: “bind: address already in use”.

Solução:
```bash
sudo ss -lntp | grep :8080
# escolha outra porta, ex.: 8081
docker rm -f lab1-web 2>/dev/null || true
docker run -d --name lab1-web -p 8081:80 lab1-nginx:1.0
```

### Container sobe e cai imediatamente
Veja logs:
```bash
docker logs lab1-web
```

Causa comum: Nginx não ficou em foreground. Confirme se o `CMD` está:
```dockerfile
CMD ["nginx", "-g", "daemon off;"]
```

### Permission denied ao usar Docker
Use `sudo` ou ajuste o grupo (pré-requisitos).

---

## Bônus (para quem terminar antes)
1) Ver camadas da imagem:
```bash
docker history lab1-nginx:1.0
```

2) Ver consumo rápido:
```bash
docker stats --no-stream
```

3) Ver IP interno do container:
```bash
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' lab1-web
```
