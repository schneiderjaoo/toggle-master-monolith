# Parte 1 — Passos na máquina local

Cobre o requisito 1 do Tech Challenge (rodar a aplicação, entender o monolito, 12 fatores) e a primeira cena do vídeo. A parte da nuvem está em [passos-aws-academy.md](passos-aws-academy.md).

## Passo 1 — Instalar o Docker

**Windows:** instalar o [Docker Desktop](https://www.docker.com/products/docker-desktop/) (backend WSL 2), abrir o aplicativo e aguardar o status *Engine running*. Os comandos deste roteiro funcionam no PowerShell, mas use `curl.exe` em vez de `curl` (que no PowerShell é um apelido de `Invoke-WebRequest`) ou rode tudo no Git Bash.

**Fedora:** repositório oficial:

```bash
sudo dnf -y install dnf-plugins-core
sudo dnf config-manager addrepo --from-repofile https://download.docker.com/linux/fedora/docker-ce.repo
sudo dnf -y install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
```

Fazer logout e login para o grupo `docker` valer. Conferir:

```bash
docker --version
docker compose version
docker run --rm hello-world
```

## Passo 2 — Subir a aplicação

Na raiz do repositório:

```bash
docker compose up --build
```

O que deve aparecer no log, nesta ordem:

1. `db` fica *healthy* (o `app` só inicia depois, por causa do `depends_on`).
2. `Aguardando o banco de dados em db:5432...` → `Banco de dados disponível!`
3. `Tabela 'flags' inicializada com sucesso.`
4. `Listening at: http://0.0.0.0:5000`

Problemas prováveis:

| Sintoma | Causa | Correção |
| :-- | :-- | :-- |
| `permission denied` ao ler `./entrypoint.sh` ou `app.py` | SELinux do Fedora bloqueando o bind mount `.:/app` | No `docker-compose.yaml`, trocar `- .:/app` por `- .:/app:z` |
| No Windows, `entrypoint.sh` falha com `illegal option` ou `not found` | Arquivo com fim de linha CRLF | O `.gitattributes` já força LF; se o clone for anterior a ele, rodar `git rm --cached -r . && git reset --hard` |
| `port is already allocated` | Porta 5000 em uso | Parar o outro processo ou mudar para `"5001:5000"` |
| `permission denied ... docker.sock` | Usuário ainda fora do grupo `docker` | Logout/login, ou usar `sudo` |

## Passo 3 — Testar a API

Em outro terminal:

```bash
curl http://localhost:5000/health

curl -X POST -H "Content-Type: application/json" \
  -d '{"name": "new-feature", "is_enabled": true}' http://localhost:5000/flags

curl http://localhost:5000/flags
curl http://localhost:5000/flags/new-feature

curl -X PUT -H "Content-Type: application/json" \
  -d '{"is_enabled": false}' http://localhost:5000/flags/new-feature
```

Casos de erro que valem mostrar no vídeo:

```bash
# 409: flag repetida
curl -i -X POST -H "Content-Type: application/json" -d '{"name": "new-feature"}' http://localhost:5000/flags
# 400: sem o campo obrigatório
curl -i -X POST -H "Content-Type: application/json" -d '{}' http://localhost:5000/flags
# 404: flag inexistente
curl -i http://localhost:5000/flags/nao-existe
```

## Passo 4 — Ver que o estado está no banco, não na aplicação

```bash
docker compose exec db psql -U user -d togglemaster -c "SELECT * FROM flags;"

docker compose restart app
curl http://localhost:5000/flags      # a flag continua lá
```

Isso demonstra os fatores 4 (backing service) e 6 (processo sem estado).

## Passo 5 — Análise do código

Com a aplicação entendida, ler [analise.md](analise.md) acompanhando o `app.py`:

- por que é um monolito e as vantagens/desvantagens para um MVP;
- a tabela dos 12 fatores, com a evidência de cada um no código;
- os pontos observados (`DELETE` adicionado por nós, sem autenticação, `/health` não testa o banco).

## Passo 6 — Encerrar

```bash
docker compose down        # mantém os dados no volume
docker compose down -v     # apaga também o volume do banco
```

## Checklist da parte local

- [ ] Docker instalado e `docker compose up --build` funcionando
- [ ] CRUD testado com `curl` (incluindo os erros 400/404/409)
- [ ] Persistência demonstrada após reiniciar o `app`
- [ ] Análise do monolito e 12 fatores revisada pelo grupo
- [ ] Trecho do vídeo gravado
