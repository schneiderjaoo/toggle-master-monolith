# Relatório de Entrega — Tech Challenge Fase 1 (ToggleMaster)

## Participantes

- <!-- TODO: nome completo e RM de cada participante -->

## Links

- **Vídeo de demonstração:** <!-- TODO -->
- **Diagrama de arquitetura:** <!-- TODO: link do diagrams.net/Miro --> (fonte em [docs/arquitetura.drawio](docs/arquitetura.drawio))
- **Documentação:**
  - [Análise do monolito e 12-Factor App](docs/analise.md)
  - [Arquitetura e custo na AWS](docs/guia-aws.md)
  - [Parte 1 — Passos na máquina local](docs/passos-local.md)
  - [Parte 2 — Passos na AWS Academy](docs/passos-aws-academy.md)

## Decisões tomadas

- **Região `us-east-1`**: menor custo e disponível no AWS Academy.
- **EC2 `t3.micro` com Amazon Linux 2023** na sub-rede pública: suficiente para o MVP e com Python 3.9, compatível com as dependências fixadas no `requirements.txt`.
- **RDS PostgreSQL `db.t3.micro` Single-AZ** em sub-rede privada e sem acesso público: mesmo SGBD do ambiente local, isolado da internet.
- **Duas sub-redes privadas em AZs diferentes**: exigência do DB Subnet Group do RDS.
- **Sem NAT Gateway**: o banco não precisa de saída para a internet e o NAT custaria mais que o restante da arquitetura.
- **Security Group do RDS aceita apenas o Security Group da EC2** na porta 5432, em vez de um intervalo de IPs.
- **SSH liberado só para um IP específico (`/32`)**.
- **Credenciais do banco por variáveis de ambiente** na sessão da EC2, fora do código e do Git.
- **Endpoint `DELETE /flags/<nome>` adicionado** ao `app.py`: o código original só criava, lia e atualizava, e o CRUD ficava incompleto.

## Desafios encontrados

<!-- TODO: preencher com o que realmente aconteceu durante o provisionamento. Pontos já identificados na análise: -->

- A tabela `flags` não é criada automaticamente fora do Docker: foi necessário executar `flask init-db` na EC2 antes de subir o Gunicorn.
- A aplicação escuta na porta 5000, então foi preciso liberá-la no Security Group além de 80/443.

## Estimativa de custos

Total estimado: **≈ US$ 27/mês** (detalhamento em [docs/guia-aws.md](docs/guia-aws.md#2-estimativa-de-custo-mensal)).

<!-- TODO: inserir o print da Calculadora de Preços da AWS -->
