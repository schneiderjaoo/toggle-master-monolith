# Arquitetura e custo na AWS — ToggleMaster (Fase 1)

Requisito 2 do Tech Challenge. O diagrama está em [arquitetura.drawio](arquitetura.drawio) (abrir em https://app.diagrams.net → *File → Open from → Device*).

## 1. Arquitetura

Região `us-east-1` (a mais barata e a disponível no AWS Academy).

| Recurso | Configuração |
| :-- | :-- |
| VPC | `togglemaster-vpc`, `10.0.0.0/16`, DNS hostnames habilitado |
| Sub-rede pública | `10.0.1.0/24` em `us-east-1a`, auto-assign de IP público |
| Sub-rede privada A | `10.0.11.0/24` em `us-east-1a` |
| Sub-rede privada B | `10.0.12.0/24` em `us-east-1b` |
| Internet Gateway | Anexado à VPC |
| Tabela de rotas pública | `0.0.0.0/0 → Internet Gateway`, associada à sub-rede pública |
| Tabela de rotas privada | Apenas a rota local, associada às duas sub-redes privadas |
| EC2 | `t3.micro`, Amazon Linux 2023, sub-rede pública, 8 GB gp3 |
| RDS | PostgreSQL, `db.t3.micro`, Single-AZ, 20 GB, **sem acesso público**, DB Subnet Group com as duas sub-redes privadas |

### Security Groups

**`togglemaster-ec2`** (entrada):

| Porta | Origem | Motivo |
| :-- | :-- | :-- |
| TCP 22 | Meu IP `/32` | SSH só de um IP específico |
| TCP 80 | `0.0.0.0/0` | HTTP |
| TCP 443 | `0.0.0.0/0` | HTTPS |
| TCP 5000 | `0.0.0.0/0` | Porta em que o Gunicorn escuta |

**`togglemaster-rds`** (entrada):

| Porta | Origem | Motivo |
| :-- | :-- | :-- |
| TCP 5432 | `togglemaster-ec2` | Só a EC2 fala com o banco |

### Decisões

- **Duas sub-redes privadas** porque o RDS exige um DB Subnet Group com sub-redes em pelo menos duas AZs, mesmo sendo Single-AZ.
- **Sem NAT Gateway.** O RDS não precisa de saída para a internet, e um NAT sozinho custaria mais do que todo o restante da arquitetura (cerca de US$ 33/mês mais tráfego).
- **Origem do SG do RDS é o SG da EC2**, não um CIDR: a regra continua válida se o IP da EC2 mudar e nenhuma outra máquina da VPC alcança o banco.
- **Single-AZ e instâncias micro**: é um MVP para validação; alta disponibilidade (Multi-AZ, load balancer, Auto Scaling) fica para as próximas fases.
- **A aplicação escuta na 5000**, então essa porta está aberta além de 80/443 pedidas no enunciado. Para servir na 80, a alternativa é um Nginx como proxy reverso na EC2.

## 2. Estimativa de custo mensal

Valores on-demand de `us-east-1`, 730 h/mês, conferidos na Calculadora de Preços da AWS: [estimativa online](https://calculator.aws/#/estimate?id=40fbee999ab3b0eb10e55fa3d4296b170c23d4fe) e [exportação em PDF](calculadora-aws.pdf).

| Item | Cálculo | US$/mês |
| :-- | :-- | --: |
| EC2 `t3.micro` Linux | 730 h × 0,0104 | 7,59 |
| EBS gp3 8 GB (EC2) | 8 × 0,08 | 0,64 |
| IPv4 público da EC2 | 730 h × 0,005 | 3,65 |
| RDS PostgreSQL `db.t3.micro` Single-AZ | 730 h × 0,018 | 13,14 |
| Armazenamento RDS 20 GB | 20 × 0,115 | 2,30 |
| VPC, sub-redes, IGW, Security Groups | sem custo | 0,00 |
| Transferência de dados | desprezível no MVP | ~0,00 |
| **Total** | | **≈ 27,32** |

Na calculadora, adicionar os serviços **Amazon EC2**, **Amazon RDS for PostgreSQL** e **Amazon VPC** (Public IPv4 Address) com esses parâmetros.

## 3. Passo a passo

- [Parte 1 — Passos na máquina local](passos-local.md)
- [Parte 2 — Passos na AWS Academy (Learner Lab)](passos-aws-academy.md)
