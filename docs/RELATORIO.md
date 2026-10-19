# Relatório técnico

## (a) Dockerfile — cada instrução e a ordem

1. `FROM python:3.12-slim` → imagem oficial do Python 3.12, versão leve.
2. `WORKDIR /app` → define a pasta de trabalho dentro do container.
3. `COPY requirements.txt .` → copia primeiro só as dependências.
4. `RUN pip install --no-cache-dir -r requirements.txt` → instala Flask, psycopg e Gunicorn sem guardar cache do pip.
5. `COPY . .` → copia o resto do código da aplicação.
6. `RUN mkdir ... && adduser ... && chown ...` → cria a pasta uploads e o usuário appuser (sem root), dono do /app.
7. `USER appuser` → roda a aplicação com esse usuário, sem privilégios de root.
8. `EXPOSE 5000` → documenta que a aplicação escuta na porta 5000.
9. `CMD ["gunicorn", "--bind", "0.0.0.0:5000", "app:app"]` → sobe o Gunicorn na forma exec.

**Cache de camadas:** copio o `requirements.txt` antes do código porque as dependências quase nunca mudam. Então, se eu mudar só o `app.py`, o Docker reaproveita as camadas da instalação e só refaz do `COPY . .` pra frente.

## (b) docker-compose.yml — por bloco

- **db**: imagem `postgres:16-alpine`; variáveis `POSTGRES_*` do `.env`; volume `dados_banco` em `/var/lib/postgresql/data`; bind mount do `init.sql` em `/docker-entrypoint-initdb.d/init.sql` (só leitura); healthcheck com `pg_isready`; restart unless-stopped.
- **web**: build de `./app`; variáveis `DB_*` e `UPLOAD_DIR`; volume `fotos` em `/app/uploads`; `depends_on` com `condition: service_healthy`; restart unless-stopped. **Não publica porta no host** — só é acessível pela rede interna.
- **nginx**: imagem `nginx:alpine`; publica `8080:80`; monta `./nginx/default.conf` em `/etc/nginx/conf.d/default.conf` (só leitura); `depends_on: web`; restart unless-stopped.
- **adminer**: imagem `adminer:4`; `profiles: ["debug"]`; publica `8081:8080`; `ADMINER_DEFAULT_SERVER: db`; `depends_on` do `db` com `condition: service_healthy`. Só sobe com `docker compose --profile debug up -d`.
- **rede_interna**: rede bridge compartilhada por todos os serviços.
- **volumes**: `dados_banco` e `fotos`, gerenciados pelo Docker (persistem no `down`, somem no `down -v`).

## (c) POST /itens com foto — do curl até o disco

1. O `curl.exe -F ...` envia um POST `multipart/form-data` para `localhost:8080`.
2. O mapeamento `8080:80` entrega a requisição ao container `nginx`.
3. O nginx faz `proxy_pass` para `http://web:5000` (nome `web` resolvido pela rede interna).
4. O Gunicorn recebe e repassa para o Flask (`app:app`).
5. A rota `@app.post("/itens")` lê `descricao` e `local`; se faltar algum, responde 400.
6. Com foto, gera o nome `uuid + extensão`.
7. Salva o arquivo em `/app/uploads` (`UPLOAD_DIR`).
8. `/app/uploads` é o ponto de montagem do volume `fotos`, então a foto fica guardada ali.
9. Abre conexão com o banco em `host=db`, porta 5432 (nome resolvido pela rede interna).
10. Faz o `INSERT INTO itens` e pega o `id`.
11. A linha fica no PostgreSQL, cujo diretório de dados está no volume `dados_banco`.
12. Responde **201** com `{"id":..., "foto":"..."}` e volta pelo mesmo caminho até o `curl.exe`.

## (d) Ciclo de vida dos volumes (Tarefa 5)

- `down` + `up -d`: os containers são recriados, mas os volumes ficam → **itens e fotos persistem**.
- `down -v`: remove os volumes → depois de subir de novo volta só **3 itens e 0 fotos**.
- Os 3 exemplos voltam porque o `init.sql` só roda quando o diretório de dados está vazio.

## (e) Desafios bônus

**1. Proxy reverso nginx (+0,4).**
O serviço `nginx` é o único que publica porta (`8080:80`). Ele recebe a requisição do host e repassa para `web:5000` usando `proxy_pass`, com o nome `web` resolvido pelo DNS da rede interna. O `web` deixou de publicar porta: sem o nginx, ele só é alcançável de dentro da rede. Arquivo: `nginx/default.conf`. Para o usuário nada muda — o acesso continua em `http://localhost:8080`.

**2. Backup do volume de fotos (+0,3).**
O backup é feito com um contêiner temporário `alpine` (`--rm`), que monta o volume `achados-perdidos_fotos` em modo **somente leitura** em `/dados` e a pasta do projeto em `/backup`. O `tar czf` grava `fotos-backup.tar.gz` no host. A restauração cria um volume novo e roda `tar xzf` com o volume em modo escrita. Usei `--mount type=volume` em vez de `-v` porque é a forma que funciona de maneira consistente no Windows/PowerShell. Comandos e saída: `docs/evidencias/backup-restauracao.txt`.

**3. Adminer em profile (+0,3).**
O serviço `adminer` tem `profiles: ["debug"]`, então o `docker compose up -d` normal **não** o inicia. Ele só sobe com `docker compose --profile debug up -d` e fica acessível em `http://localhost:8081`, já apontando para o servidor `db`. Detalhe importante: como ele tem profile, um `docker compose down` sem a flag não o remove; para derrubar tudo é preciso `docker compose --profile debug down`. Evidência: `docs/evidencias/adminer-profile.txt`.
