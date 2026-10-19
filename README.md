# Achados & Perdidos — Containerização (G1 Computação em Nuvem)

## Objetivo

API web Flask com PostgreSQL e armazenamento de fotos dos itens cadastrados.
O ambiente completo (aplicação + banco) sobe com **um único comando**, sem
instalar nada além do Docker, e os dados e fotos **persistem** mesmo depois
de recriar os contêineres.

## Pré-requisitos

- Docker Desktop (Windows) com Docker Compose v2;
- Porta `8080` livre no host;
- No PowerShell do Windows, usar `curl.exe` (o `curl` é alias do `Invoke-WebRequest`).

## Configurar o ambiente (.env)

Na pasta `achados-perdidos/`, copie o exemplo e ajuste a senha:

```powershell
copy .env.example .env
```

O `.env` define 6 variáveis, lidas pelo `docker-compose.yml`:

| Variável | Usada por | Valor padrão |
| --- | --- | --- |
| `POSTGRES_DB` | serviço `db` | `achados` |
| `POSTGRES_USER` | serviço `db` | `achados` |
| `POSTGRES_PASSWORD` | serviço `db` | `senha-local` |
| `DB_NAME` | serviço `web` | `achados` |
| `DB_USER` | serviço `web` | `achados` |
| `DB_PASSWORD` | serviço `web` | `senha-local` |

`POSTGRES_*` e `DB_*` devem ter **os mesmos valores** (um define o banco, o
outro conecta a ele). O `.env` está no `.gitignore` e **não** é versionado.

## Subir o ambiente

```powershell
docker compose up -d --build
```

Verifique que os dois serviços estão no ar e o banco saudável:

```powershell
docker compose ps
```

Esperado: `db` como **healthy**, `nginx` e `web` como **running**. A porta
`8080` é publicada pelo **nginx**, que encaminha para o `web` na rede interna
(o `web` não publica porta no host).

Teste a aplicação:

```powershell
curl.exe http://localhost:8080/health
```

Resposta esperada: `"status": "ok"` e `"banco": "conectado"`.

## Testar a API

```powershell
curl.exe http://localhost:8080/itens
curl.exe "http://localhost:8080/itens?local=biblio"
curl.exe -F descricao="Teste" -F local="Sala" -F foto=@docs\evidencias\foto-teste07.png http://localhost:8080/itens
curl.exe http://localhost:8080/estatisticas
```

O terceiro comando cadastra um item com foto; a URL da foto aparece na
resposta e em `GET /itens`. As saídas dos testes estão em `docs/evidencias/`.

## Parar o ambiente (mantém dados)

```powershell
docker compose down
```

Containers e rede saem, mas os volumes `dados_banco` e `fotos` continuam:
na próxima subida os itens e fotos estão lá.

## Destruir o ambiente (remove volumes)

```powershell
docker compose down -v
```

Apaga também os volumes. Na próxima subida, o `init.sql` roda de novo e
voltam apenas os 3 itens de exemplo (sem fotos).

## Painel do banco — Adminer (opcional)

O Adminer está em um *profile* `debug`, então não sobe no comando normal.
Para subir com ele e abrir o painel em `http://localhost:8081`:

```powershell
docker compose --profile debug up -d
```

Para derrubar tudo, incluindo o Adminer:

```powershell
docker compose --profile debug down
```

## Backup e restauração do volume de fotos

O volume `achados-perdidos_fotos` pode ser salvo em um `.tar.gz` com um
contêiner temporário (o volume é montado em modo somente leitura):

```powershell
docker run --rm --mount "type=volume,src=achados-perdidos_fotos,dst=/dados,readonly" --mount "type=bind,src=$((Get-Location).Path),dst=/backup" alpine sh -c "tar czf /backup/fotos-backup.tar.gz -C /dados ."
```

Para restaurar em um volume novo:

```powershell
docker volume create achados-perdidos_fotos_restaurado
docker run --rm --mount "type=volume,src=achados-perdidos_fotos_restaurado,dst=/dados" --mount "type=bind,src=$((Get-Location).Path),dst=/backup,readonly" alpine sh -c "tar xzf /backup/fotos-backup.tar.gz -C /dados"
```

A demonstração completa está em `docs/evidencias/backup-restauracao.txt`.

## Rotas da API

| Método | Rota | Descrição |
| --- | --- | --- |
| GET | `/health` | Status da aplicação e do banco |
| GET | `/itens` | Lista itens; `?local=texto` filtra por local |
| POST | `/itens` | Cadastra item (`descricao`, `local`, `foto` opcional) |
| GET | `/fotos/<nome>` | Retorna a foto armazenada |
| GET | `/estatisticas` | Total de itens no banco e de fotos no volume |

## Documentação complementar

- `docs/arquitetura.md` — diagrama de arquitetura (rede, volumes, portas);
- `docs/RELATORIO.md` — relatório técnico (Dockerfile, Compose, fluxo de uma requisição, volumes);
- `docs/evidencias/` — saídas dos testes das Tarefas 4 e 5, com legendas;
- `docs/REFLEXAO.md` — questões de reflexão e equivalência com serviços de nuvem.
