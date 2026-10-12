# Análise da aplicação — ToggleMaster (Fase 1)

Requisito 1 do Tech Challenge: por que a aplicação é um monolito, vantagens e desvantagens para um MVP, e avaliação frente aos 12 Fatores.

## 1. Como a aplicação funciona

| Arquivo | Papel |
| :-- | :-- |
| `app.py` | Toda a aplicação: rotas HTTP (Flask), validação, regras, acesso ao PostgreSQL (`psycopg2`) e criação da tabela `flags` (`flask init-db`). |
| `requirements.txt` | Dependências Python com versão fixa (Flask, Werkzeug, psycopg2-binary, gunicorn). |
| `Dockerfile` | Imagem `python:3.9-slim` com as dependências e o `postgresql-client` (para o `pg_isready`). |
| `entrypoint.sh` | Espera o banco responder, roda `flask init-db` e sobe o Gunicorn na porta 5000. |
| `docker-compose.yaml` | Ambiente local: serviço `app` + serviço `db` (`postgres:13`) com volume persistente. |

Fluxo de uma requisição: Gunicorn (porta 5000) → rota Flask → abre uma conexão com o PostgreSQL → executa o SQL → fecha a conexão → responde JSON.

Endpoints: `GET /health`, `POST /flags`, `GET /flags`, `GET /flags/<nome>`, `PUT /flags/<nome>`.

## 2. Por que é um monolito

- **Uma única unidade de deploy.** Existe um artefato (um processo Gunicorn / uma imagem). Qualquer mudança, por menor que seja, republica a aplicação inteira.
- **Todas as responsabilidades no mesmo processo e no mesmo arquivo.** `app.py` concentra a camada HTTP, a validação de entrada, a regra de negócio, o acesso a dados e até a criação do schema. Não há fronteiras entre módulos.
- **Um único banco de dados compartilhado**, acessado diretamente por todas as rotas.
- **Escala como um bloco.** Não dá para escalar só a leitura de flags (que tende a ser o caminho mais quente) separada da escrita/administração.
- **Uma única stack e um único ciclo de vida**: mesma linguagem, mesmas dependências, mesma versão para tudo.

### Vantagens para um MVP

- **Velocidade de entrega**: um repositório, um deploy, um banco. É o caminho mais curto para ter feedback.
- **Simplicidade operacional**: uma EC2 e um RDS bastam; não há service discovery, mensageria nem orquestrador.
- **Fácil de entender e depurar**: o fluxo inteiro cabe em um arquivo; um erro tem um único lugar para ser procurado.
- **Sem chamadas de rede entre componentes**: sem latência extra e sem falhas parciais de sistema distribuído.
- **Custo baixo** de infraestrutura e de time.

### Desvantagens

- **Escalabilidade grosseira**: só escala replicando o bloco inteiro.
- **Ponto único de falha**: um bug ou vazamento de recursos derruba todas as funcionalidades de uma vez.
- **Acoplamento cresce com o tempo**: sem fronteiras internas, novas features (segmentação de usuários, auditoria, autenticação) tendem a se entrelaçar.
- **Deploy tudo-ou-nada**: qualquer alteração exige republicar tudo, aumentando o risco de cada release.
- **Dificulta times paralelos**: várias equipes mexendo no mesmo código e no mesmo schema.
- **Presa a uma stack**: não é possível adotar outra tecnologia só para uma parte.

Para a Fase 1 a troca é favorável: o domínio é pequeno e a prioridade é validar a ideia. As desvantagens passam a pesar nas fases seguintes, quando o projeto evolui para um sistema distribuído.

## 3. 12-Factor App aplicado ao projeto

| # | Fator | Situação | Evidência no projeto | Ajuste para produção |
| :-: | :-- | :-: | :-- | :-- |
| 1 | Codebase | ✅ Atende | Um repositório Git, uma base de código para todos os ambientes. | — |
| 2 | Dependencies | ✅ Atende (parcial) | `requirements.txt` com versões fixas; isolamento por `venv` ou imagem Docker. | Travar também as dependências transitivas (`pip-tools`/`pip freeze`). O `pg_isready` é uma dependência de sistema implícita do `entrypoint.sh`. |
| 3 | Config | ✅ Atende (parcial) | `DB_HOST`, `DB_NAME`, `DB_USER`, `DB_PASSWORD` lidos de variáveis de ambiente em `app.py`. | A senha está em texto puro no `docker-compose.yaml` (aceitável só em dev). `DB_PORT` é exigida pelo `entrypoint.sh` mas ignorada pelo `app.py`. Em produção, usar Secrets Manager / SSM Parameter Store. |
| 4 | Backing services | ✅ Atende | O banco é um recurso anexado por configuração: trocar o Postgres local pelo RDS não exige mudar código. | — |
| 5 | Build, release, run | ⚠️ Parcial | Existe `Dockerfile`, mas na EC2 o deploy é `git clone` + `pip install` na própria máquina; não há artefato versionado nem release imutável. O compose monta `.:/app`, misturando código e execução. | Pipeline de CI que gera imagem versionada e um release = imagem + config. |
| 6 | Processes | ✅ Atende | Processo sem estado: nada em memória ou disco entre requisições; tudo persiste no PostgreSQL. | — |
| 7 | Port binding | ✅ Atende | A aplicação é autocontida e exporta HTTP via Gunicorn em `0.0.0.0:5000`. | — |
| 8 | Concurrency | ⚠️ Parcial | Por ser stateless, pode escalar horizontalmente. Mas o Gunicorn sobe com 1 worker síncrono e cada requisição abre uma conexão nova com o banco. | Definir `--workers`, usar pool de conexões e colocar um load balancer na frente de várias instâncias. |
| 9 | Disposability | ⚠️ Parcial | Sobe rápido e o Gunicorn encerra de forma graciosa com SIGTERM (`exec` no entrypoint). Na EC2, porém, roda em foreground na sessão SSH: fechar o terminal derruba a aplicação e nada a reinicia. | Rodar como serviço (`systemd`) ou contêiner com política de restart. |
| 10 | Dev/prod parity | ⚠️ Parcial | Mesmo código e mesmo SGBD (PostgreSQL). Mas local é Docker + `postgres:13`, e na AWS é `venv` direto na EC2 + RDS (versão possivelmente diferente). | Usar a mesma imagem Docker em todos os ambientes e alinhar a versão do Postgres. |
| 11 | Logs | ⚠️ Parcial | Escreve em stdout (`print`, `PYTHONUNBUFFERED=1`), sem gerenciar arquivos de log. Porém sem estrutura/níveis, o access log do Gunicorn fica desligado e, na EC2, a saída morre no terminal. | Módulo `logging` com formato estruturado, `--access-logfile -` e envio para o CloudWatch Logs. |
| 12 | Admin processes | ✅ Atende (parcial) | `flask init-db` é um comando pontual, no mesmo código e com a mesma config. | O entrypoint o executa a cada start com `\|\| true`, escondendo falhas. Usar ferramenta de migração (Alembic/Flyway) como etapa explícita de release. |

**Resumo:** atende bem a 1, 2, 3, 4, 6, 7 e 12; precisa de ajustes em 5, 8, 9, 10 e 11 para um ambiente de produção robusto.

## 4. Outros pontos observados no código

- **Não existe endpoint `DELETE`.** A proposta fala em "criar, ler, atualizar e deletar", mas `app.py` só implementa criar, ler e atualizar.
- **A API não tem autenticação**: qualquer um que alcance a porta pode criar e alterar flags.
- **`/health` não testa o banco**: responde `ok` mesmo com o RDS inacessível.
- **A tabela não é criada automaticamente fora do Docker.** Na EC2 é preciso rodar `flask init-db` uma vez, senão toda rota de `/flags` retorna erro 500 (o README não menciona esse passo).
- As respostas de erro 500 devolvem `details` com a mensagem da exceção, o que pode expor informação interna.
- As mensagens de sucesso no README estão em inglês, mas o código responde em português.
