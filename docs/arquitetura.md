# Diagrama de arquitetura

Visão do ambiente: host, portas publicadas, contêineres, rede interna, volumes
e bind mount. Rede confirmada: `achados-perdidos_rede_interna`.

## Visão geral (Mermaid)

```mermaid
flowchart LR
    host["Host (navegador / curl)"] -- "porta 8080" --> nginx

    subgraph rede_interna["achados-perdidos_rede_interna (bridge) - rede privada"]
        nginx["contêiner nginx<br/>nginx:alpine<br/>80 (proxy reverso)"]
        web["contêiner web<br/>python:3.12-slim + gunicorn<br/>0.0.0.0:5000 (SEM porta no host)"]
        db["contêiner db<br/>postgres:16-alpine<br/>5432 (NÃO publicada)"]
        adminer["contêiner adminer<br/>profile debug (opcional)<br/>8080 interno"]
        nginx -- "proxy_pass http://web:5000" --> web
        web -- "host: db (DNS do Docker)" --> db
        adminer -. "host: db (só com --profile debug)" .-> db
    end

    web --- fotos[("volume achados-perdidos_fotos<br/>montado em /app/uploads")]
    db --- dados[("volume achados-perdidos_dados_banco<br/>montado em /var/lib/postgresql/data")]
    db --- initsql["bind mount: ./db/init.sql → /docker-entrypoint-initdb.d/init.sql (ro)"]
    adminer --- adminerPorta["porta publicada: 8081:8080<br/>(só com --profile debug)"]
    foraDaRede["FORA da rede interna:<br/>host, portas publicadas (8080:80 e 8081:8080), volumes e bind mount"]
```

## O que fica dentro e fora da rede interna

| Componente | Dentro da rede? | Observação |
| --- | --- | --- |
| Contêiner `nginx` | Sim | Único ponto de entrada da aplicação; publica `8080:80` e encaminha para `web:5000` |
| Contêiner `web` | Sim | Não publica porta no host: só é acessível pelo nginx, via rede interna |
| Contêiner `db` | Sim | Acessível apenas como `db` de dentro da rede; sem porta no host |
| Contêiner `adminer` | Sim (opcional) | Sobe só com `--profile debug`; publica `8081:8080` |
| Volumes (`dados_banco`, `fotos`) | Fora | São recursos do engine Docker, montados nos contêineres |
| Bind mount `init.sql` | Fora | Arquivo do host, somente leitura |
