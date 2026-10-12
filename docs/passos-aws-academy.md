# Parte 2 — Passos na AWS Academy (Learner Lab)

Cobre os requisitos 2 e 3 do Tech Challenge. A arquitetura e o custo estão em [guia-aws.md](guia-aws.md); o diagrama em [arquitetura.drawio](arquitetura.drawio). A parte local está em [passos-local.md](passos-local.md).

## O que muda por ser conta de estudante

Conferir estes pontos no *Readme* do próprio Learner Lab, pois as regras variam entre turmas:

| Restrição | Efeito no projeto |
| :-- | :-- |
| Sessão de laboratório com tempo limitado (em geral 4 h) | Ao encerrar, a EC2 é parada. Ao voltar, é preciso reexportar as variáveis e subir o Gunicorn de novo. |
| IP público da EC2 muda a cada parada/início | O endereço de teste muda; atualizar os `curl` e o vídeo. |
| Orçamento fixo de créditos (em geral US$ 50) | Estourar o orçamento bloqueia a conta. Parar o RDS quando não estiver usando e apagar tudo após a avaliação. |
| Só as regiões `us-east-1` e `us-west-2` | Usar `us-east-1`. |
| Não é possível criar usuários/roles no IAM | Existe a role pronta `LabRole`. No RDS, **desmarcar Enhanced Monitoring**, que tentaria criar uma role. |
| Key pair `vockey` já criado | A chave privada é baixada em *AWS Details → Download PEM* (`labsuser.pem`). |
| Tipos de instância limitados (até `large` na EC2, até `medium` no RDS) | `t3.micro` e `db.t3.micro` estão dentro do limite. |

A Calculadora de Preços é pública e não depende da conta do laboratório.

## Passo 1 — Iniciar o laboratório

1. AWS Academy → curso → *Modules* → *Learner Lab* → **Start Lab**.
2. Aguardar o círculo ao lado de **AWS** ficar verde e clicar nele para abrir o console.
3. Conferir a região **N. Virginia (us-east-1)** no canto superior direito.
4. Em **AWS Details** → **Download PEM** e guardar o `labsuser.pem` fora do repositório.

## Passo 2 — Rede

1. **VPC → Create VPC → VPC only**: nome `togglemaster-vpc`, CIDR `10.0.0.0/16`. Depois *Actions → Edit VPC settings* → marcar *Enable DNS hostnames*.
2. **Subnets → Create subnet** (três, na VPC criada):

   | Nome | AZ | CIDR |
   | :-- | :-- | :-- |
   | `togglemaster-public-a` | us-east-1a | `10.0.1.0/24` |
   | `togglemaster-private-a` | us-east-1a | `10.0.11.0/24` |
   | `togglemaster-private-b` | us-east-1b | `10.0.12.0/24` |

   Na pública: *Actions → Edit subnet settings → Enable auto-assign public IPv4 address*.
3. **Internet gateways → Create**: `togglemaster-igw` → *Actions → Attach to VPC*.
4. **Route tables**:
   - `togglemaster-rt-public`: adicionar a rota `0.0.0.0/0 → togglemaster-igw` e associar à sub-rede pública.
   - `togglemaster-rt-private`: só a rota local; associar às duas sub-redes privadas.

## Passo 3 — Security Groups

Criar os dois na `togglemaster-vpc`, nesta ordem.

**`sg-togglemaster-ec2`** — regras de entrada:

| Tipo | Porta | Origem |
| :-- | :-- | :-- |
| SSH | 22 | *My IP* |
| HTTP | 80 | `0.0.0.0/0` |
| HTTPS | 443 | `0.0.0.0/0` |
| Custom TCP | 5000 | `0.0.0.0/0` |

**`sg-togglemaster-rds`** — regra de entrada:

| Tipo | Porta | Origem |
| :-- | :-- | :-- |
| PostgreSQL | 5432 | `sg-togglemaster-ec2` (selecionar o grupo, não um IP) |

> Se o seu IP mudar (outra rede, outro dia), o SSH para de conectar: editar a regra e escolher *My IP* de novo.

## Passo 4 — Banco de dados RDS

1. **RDS → Subnet groups → Create**: `togglemaster-db-subnets`, VPC criada, AZs `us-east-1a` e `us-east-1b`, as duas sub-redes privadas.
2. **RDS → Databases → Create database**:
   - *Standard create*, **PostgreSQL**, template *Free tier* (ou *Dev/Test*, Single-AZ).
   - Identificador `togglemaster-db`; usuário master e senha próprios (*Self managed*). Anotar fora do repositório.
   - Classe `db.t3.micro`; storage 20 GB; desmarcar *storage autoscaling*.
   - *Connectivity*: *Don't connect to an EC2 compute resource*, VPC `togglemaster-vpc`, subnet group `togglemaster-db-subnets`, **Public access: No**, SG existente `sg-togglemaster-rds` (remover o `default`).
   - *Monitoring*: **desmarcar Enhanced Monitoring**.
   - *Additional configuration*: **Initial database name = `togglemaster`**. Sem isso o banco não é criado.
3. Aguardar o status *Available* (5 a 10 minutos) e copiar o **Endpoint**.

## Passo 5 — Instância EC2

**EC2 → Launch instance**:

- Nome `togglemaster-app`; AMI **Amazon Linux 2023**; tipo `t3.micro`.
- Key pair: **vockey**.
- *Network settings → Edit*: VPC `togglemaster-vpc`, sub-rede `togglemaster-public-a`, *Auto-assign public IP: Enable*, SG existente `sg-togglemaster-ec2`.
- Storage: 8 GB gp3.

> Usar Amazon Linux 2023 (Python 3.9). O `psycopg2-binary==2.9.5` do `requirements.txt` não tem pacote pronto para Python 3.12+, então no Ubuntu 24.04 o `pip install` falha.

## Passo 6 — Conectar por SSH e instalar

Na máquina local:

```bash
chmod 400 ~/labsuser.pem
ssh -i ~/labsuser.pem ec2-user@<ip-publico-da-ec2>
```

Na EC2:

```bash
sudo dnf update -y
sudo dnf install -y git python3 python3-pip

git clone <url-do-seu-repositorio>
cd toggle-master-monolith

python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
```

## Passo 7 — Configurar as credenciais e rodar

O `read -s` evita que a senha fique no histórico do shell:

```bash
export DB_HOST='<endpoint-do-rds>'
export DB_NAME='togglemaster'
export DB_USER='<usuario-master>'
read -s -p 'Senha do banco: ' DB_PASSWORD && export DB_PASSWORD
```

Criar a tabela (uma única vez) e subir a aplicação:

```bash
flask init-db      # deve imprimir: Tabela 'flags' inicializada com sucesso.
gunicorn --bind 0.0.0.0:5000 app:app
```

> O `flask init-db` não está no README original, mas é obrigatório fora do Docker: sem ele as rotas de `/flags` retornam 500. Também serve de teste de conexão com o RDS — se travar e der *timeout*, revisar o SG do RDS e o subnet group.

## Passo 8 — Validar publicamente

Da máquina local:

```bash
EC2=http://<ip-ou-dns-publico-da-ec2>:5000

curl $EC2/health
curl -X POST -H "Content-Type: application/json" -d '{"name": "new-feature", "is_enabled": true}' $EC2/flags
curl $EC2/flags
curl -X PUT -H "Content-Type: application/json" -d '{"is_enabled": false}' $EC2/flags/new-feature
curl $EC2/flags/new-feature
```

## Passo 9 — Evidências para o vídeo

- Parar o Gunicorn (`Ctrl+C`), subir de novo e mostrar que `GET /flags` ainda devolve a flag: o dado está no RDS.
- Da máquina local, `nc -zv -w 5 <endpoint-do-rds> 5432` não conecta: o banco é privado.
- No console, as regras de entrada dos dois Security Groups.
- Na EC2, `env | grep DB_ | sed 's/PASSWORD=.*/PASSWORD=****/'`: credenciais em variáveis de ambiente, fora do código e do Git.

## Passo 10 — Custo e diagrama

1. Montar a estimativa em https://calculator.aws/ com os itens de [guia-aws.md](guia-aws.md#2-estimativa-de-custo-mensal) e tirar o print.
2. Abrir [arquitetura.drawio](arquitetura.drawio) em https://app.diagrams.net, salvar no Google Drive/OneDrive e gerar o link de compartilhamento.
3. Preencher os `TODO` do [ENTREGA.md](../ENTREGA.md).

## Passo 11 — Entre sessões e ao final

**Ao retomar o laboratório** (nova sessão): *Start Lab* → conferir o novo IP público da EC2 → iniciar o RDS se estiver parado → SSH → `cd toggle-master-monolith && source venv/bin/activate` → reexportar as variáveis do passo 7 → `gunicorn ...`. O `flask init-db` não precisa ser repetido.

**Para poupar créditos**: *RDS → Actions → Stop temporarily* quando não estiver usando.

**Depois da avaliação**, apagar nesta ordem: instância EC2 → banco RDS (sem snapshot final) → DB subnet group → Security Groups (primeiro o do RDS) → sub-redes → tabelas de rotas → Internet Gateway (detach e delete) → VPC.

## Checklist da parte AWS

- [ ] VPC, 3 sub-redes, IGW e tabelas de rotas
- [ ] 2 Security Groups com as regras corretas
- [ ] RDS PostgreSQL *Available*, sem acesso público
- [ ] EC2 acessível por SSH
- [ ] Aplicação respondendo pelo IP público
- [ ] Print da calculadora e link do diagrama
- [ ] Vídeo gravado e `ENTREGA.md` preenchido
- [ ] Recursos removidos após a avaliação
