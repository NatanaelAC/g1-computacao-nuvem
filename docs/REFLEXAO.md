# Reflexão e equivalência em nuvem

## Q1 — Por que o web se conecta pelo nome `db`?

No `docker-compose.yml`, `web` e `db` estão na mesma rede `rede_interna`, e o Compose habilita o DNS interno do Docker. Com isso, o nome `db` é resolvido para o IP do container do banco, e o `app.py` conecta usando `host=db`. Se eu usasse `docker run` na rede bridge padrão, teria que passar o IP na mão.

## Q2 — Volume nomeado vs. bind mount

Volume nomeado é gerenciado pelo Docker e fica em `/var/lib/docker/volumes`. Bind mount é uma pasta ou arquivo do meu host. Usei bind mount no `./db/init.sql` porque ele faz parte do repositório e eu quero ler exatamente aquele arquivo. Já os dados do Postgres ficam no volume `dados_banco`, que é isolado e persiste no `down`.

## Q3 — Alterar o init.sql depois que o ambiente já subiu

Não, a mudança não é aplicada. O `init.sql` só roda quando o diretório de dados está vazio, e o volume `dados_banco` já tem dados. Pra aplicar sem perder dados, o jeito é rodar o SQL dentro do container já em execução (com `psql`) ou usar uma ferramenta de migração.

## Q4 — depends_on sem service_healthy

`depends_on` sozinho só ordena a subida: o container `db` pode estar "rodando" mas ainda não aceitar conexões. Com `condition: service_healthy` o `web` só sobe quando o healthcheck (`pg_isready`) confirma que o banco está pronto, evitando erro na primeira conexão.

## Q5 — Ordem do COPY e cache de camadas

Copio o `requirements.txt` e instalo as dependências antes de copiar o código, pra aproveitar o cache de camadas. Se eu mudar só o `app.py`, o Docker reaproveita a instalação das dependências e só refaz do `COPY . .` pra frente. Se copiasse tudo antes, qualquer mudança reinstalaria tudo e o build ficaria bem mais lento.

## Q6 — Por que a 5432 não é publicada + LGPD

Deixei a porta 5432 sem publicação (sem `ports` no `db`), então o banco só é acessível dentro da `rede_interna`, pelo nome `db` — apenas o `nginx` publica `8080:80` e encaminha para o `web`. Isso protege dados sensíveis (LGPD), como documentos que podem aparecer nas fotos. Duas medidas a mais: um proxy/autenticação na porta 8080 e guardar os segredos num gerenciador de segredos na nuvem.

## Q7 — docker compose up -d --scale web=3

`--scale web=3` sobe 3 containers web, mas todos tentariam usar a porta 8080 do host e daria conflito. Pra rodar 3 réplicas precisa de um balanceador (tipo Nginx) na frente distribuindo as requisições. O volume `fotos` pode ser compartilhado pelas réplicas; o único cuidado é não gravar o mesmo arquivo ao mesmo tempo. O projeto já tem um nginx como proxy reverso (desafio bônus — ver `docs/RELATORIO.md`, seção (e)), que seria o ponto de partida para esse balanceamento.

## 5.1 Do ambiente local para a nuvem

| Peça do ambiente local | Microsoft Azure | AWS | Google Cloud |
| --- | --- | --- | --- |
| Contêiner web | Azure App Service / ACI | AWS App Runner / ECS | Cloud Run |
| Contêiner db + volume dados_banco | Azure Database for PostgreSQL | Amazon RDS for PostgreSQL | Cloud SQL for PostgreSQL |
| Volume fotos | Azure Blob Storage | Amazon S3 | Google Cloud Storage |
| rede_interna | Azure VNet | Amazon VPC | Google VPC |
| Arquivo .env (segredos) | Azure Key Vault | AWS Secrets Manager | Google Secret Manager |
| Imagem construída localmente | Azure Container Registry (ACR) | Amazon ECR | Artifact Registry |

**Parágrafo final:** a peça que muda mais na migração é o **banco + seu volume**. Localmente é um container com volume; na nuvem virou serviço gerenciado (RDS, Cloud SQL ou Azure Database), que já cuida de backup e disponibilidade. As fotos também saem do volume e vão pra um object storage.
