# LogFree

SoluÃ§Ã£o local por cidade que indica, para frotas de Ãºltima milha (moto, carro, caminhÃ£o leve), o posto de combustÃ­vel com **melhor custo real por km rodado** â€” nÃ£o apenas o preÃ§o mais baixo na bomba.

Acesso por **bot Telegram** (motorista) e **web** (gestor/operador).

---

## Como rodar (passo a passo)

### 1. Setup local (uma vez)

```bash
cd ~/projetos/logfree
python3 -m venv .venv
source .venv/bin/activate
pip install -e ".[dev]"
cp .env.example .env
```

Pule o `python3 -m venv .venv` se jÃ¡ tiver `.venv` no projeto.

### 2. DB + seed

```bash
alembic upgrade head           # cria tabelas + triggers
python -m app.db.seed          # popula cidade_demo (15 postos, 6 combustÃ­veis)
logfree bootstrap-admin --email admin@local --password troca123-em-prod --cidade-id 1
```

### 3. Sobe o servidor

```bash
uvicorn app.api.main:app --host 0.0.0.0 --port 8000 --reload
```

Em outra aba, valide:

```bash
curl http://localhost:8000/internal/ready
# {"status":"ok"}

curl http://localhost:8000/internal/metrics
```

### 4. Bate na API

```bash
# Esperado 401: precisa de auth
curl -X POST http://localhost:8000/melhor-posto \
  -H 'Content-Type: application/json' \
  -d '{
    "lat": -12.97, "lng": -38.50,
    "combustivel": "gasolina_comum",
    "autonomia_restante_km": 200,
    "modo_abastecimento": "completo",
    "top_n": 3
  }'

# Com bot service token (motorista demo do seed: telegram_id=100001)
curl -X POST http://localhost:8000/melhor-posto \
  -H 'Content-Type: application/json' \
  -H 'X-Bot-Token: dev-bot-token-change-me' \
  -H 'X-Bot-User-Telegram-Id: 100001' \
  -d '{
    "lat": -12.97, "lng": -38.50,
    "combustivel": "gasolina_comum",
    "autonomia_restante_km": 200,
    "modo_abastecimento": "completo",
    "top_n": 3
  }' | jq .
```

### 5. Admin web

Abra <http://localhost:8000/admin/login> no browser, entre com `admin@local` + a senha. VocÃª vÃª as cidades, lista de postos com formulÃ¡rio HTMX para registrar preÃ§o, mÃ©tricas. Login errado, CSRF ausente e cross-city sÃ£o bloqueados (testes cobrem).

### 6. Testes

```bash
pytest -q                                   # todos (~98)
pytest tests/unit -q                        # sÃ³ unitÃ¡rios (rÃ¡pido)
pytest tests/integration -q                 # API + admin + bot
pytest --cov=app --cov-report=term-missing  # com cobertura
ruff check app tests                        # lint
```

### 7. CLI Ãºteis

```bash
logfree audit-verify                                            # checa cadeia hash do evento_audit
logfree audit-unblock --approver=admin@x --motivo="..."         # break-glass (precisa AUDIT_UNBLOCK_KEY)
logfree flag-set --chave=ranking.osrm_enabled --valor=true
logfree flag-set --chave=kill_switch.global --valor=true        # desliga /melhor-posto (503)
logfree replay-tombstones                                       # pÃ³s-restore: re-anonimiza usuÃ¡rios tombstoned
```

### 8. Bot Telegram (opcional, sÃ³ com token real)

NÃ£o funciona sem token real. Se quiser testar:

```bash
# em .env
TG_BOT_TOKEN=<seu-token-do-BotFather>
TG_WEBHOOK_SECRET=$(openssl rand -hex 32)

# expor pra fora (ngrok)
ngrok http 8000

# registrar webhook (use o MESMO segredo no path e no header)
curl "https://api.telegram.org/bot<TOKEN>/setWebhook" \
  -d "url=https://<seu-host-ngrok>/bot/webhook/$TG_WEBHOOK_SECRET" \
  -d "secret_token=$TG_WEBHOOK_SECRET"
```

Sem fazer isso, o webhook responde 503. Os fluxos do bot estÃ£o cobertos por testes mockando o JSON do Telegram.

---

### Endpoints principais

- `POST /melhor-posto` â€” auth obrigatÃ³ria (sessÃ£o admin OU `X-Bot-Token` + `X-Bot-User-Telegram-Id`).
- `POST /bot/webhook/{path_secret}` â€” webhook Telegram (header `X-Telegram-Bot-Api-Secret-Token` validado).
- `GET /admin/login`, `GET /admin/`, `GET /admin/postos/{cidade_id}` â€” UI HTMX.
- `GET /internal/ready`, `GET /internal/metrics` â€” healthcheck e mÃ©tricas.

### Erros comuns

| Sintoma | Causa | SoluÃ§Ã£o |
|---|---|---|
| `no such table: alembic_version` | migration nÃ£o rodou | `alembic upgrade head` |
| `combustivel desconhecido` na API | falta seed da 2a migration | `alembic upgrade head` (gera os 6 combustÃ­veis) |
| `flock ocupado` ao iniciar uvicorn | outro processo segurando o owner lease | `pkill -f uvicorn` (ou cheque o PID que estÃ¡ com o lock) |
| `401` em `/melhor-posto` com bot token | `X-Bot-Token` errado, `X-Bot-User-Telegram-Id` inexistente, ou usuÃ¡rio sem `opt_in_lgpd=1` | Confira `.env` e `usuario.opt_in_lgpd`; o motorista demo do seed (`telegram_id=100001`) jÃ¡ vem com opt-in. |
| `403` em `/melhor-posto` | sem opt-in LGPD | bot deve mandar `/aceitar_termos` antes de `/melhor` |

VariÃ¡veis de ambiente em `.env.example`. Em produÃ§Ã£o, configure pela plataforma (Fly secrets / Railway env).

Plano completo: ver [`PLAN.md`](PLAN.md) (3 rodadas de revisÃ£o adversarial â€” engenharia, seguranÃ§a, ops/SRE).

## Runbooks

[`runbook/README.md`](runbook/README.md) lista todos os runbooks (rollback, restore, audit chain break, OSRM degradaÃ§Ã£o, segredos, capacidade, etc).

---

## 1. Problema

Motoristas de Ãºltima milha perdem margem abastecendo no posto errado. A decisÃ£o Ã© tomada pelo preÃ§o da placa, mas o que importa Ã© o **custo efetivo por km**, que depende de:

- preÃ§o do litro
- consumo do veÃ­culo naquele combustÃ­vel
- desvio que o motorista faz pra chegar atÃ© o posto

Hoje ninguÃ©m entrega isso. Waze e Google mostram rota. Apps de preÃ§o mostram preÃ§o. Nenhum junta os dois com o perfil do veÃ­culo e o contexto de entrega urbana.

## 2. Proposta de valor

> "O posto mais barato **pra vocÃª, hoje, considerando seu veÃ­culo e onde vocÃª estÃ¡.**"

Diferenciais:

- Curadoria **local por cidade** â€” dados confiÃ¡veis, sem crowdsourcing caÃ³tico.
- CÃ¡lculo de **custo efetivo por km** por veÃ­culo, nÃ£o preÃ§o bruto.
- Acesso por **Telegram**, onde o motorista jÃ¡ vive, e **web** pro gestor da frota.
- Pensado pra **Ãºltima milha**: raio curto, muitas paradas, decisÃ£o rÃ¡pida.

## 3. Escopo do MVP

**Entra:**

- Cadastro manual de postos e preÃ§os (operador da cidade piloto).
- Cadastro de veÃ­culos (tipo, modelo, combustÃ­vel, km/l).
- Bot Telegram respondendo:
  - `/postos` â€” lista os N postos mais baratos por combustÃ­vel na cidade.
  - `/melhor <combustÃ­vel>` â€” melhor posto pro veÃ­culo do usuÃ¡rio, considerando localizaÃ§Ã£o enviada.
  - `/rota origem â†’ destino` â€” rota sugerida + posto recomendado no caminho.
- Web: mapa da cidade com postos, tabela ordenÃ¡vel, painel admin pra atualizar preÃ§os.
- RelatÃ³rio simples por motorista/frota (economia estimada).

**Fica pra depois:** IA, previsÃ£o de preÃ§os, integraÃ§Ã£o com cartÃ£o-frota, OBD-II, multi-cidade, app nativo.

## 4. Fluxos principais

**Motorista (Telegram):**

1. `/start` â€” cadastra veÃ­culo (tipo + km/l mÃ©dio).
2. Envia localizaÃ§Ã£o.
3. `/melhor diesel` â€” bot consulta DB, calcula custo efetivo, devolve top 3 postos com preÃ§o, distÃ¢ncia, desvio e economia estimada vs. o mais prÃ³ximo.

**Gestor (Web):**

1. Login admin, tela de postos, edita preÃ§o do dia.
2. Dashboard: mapa, lista, histÃ³rico de preÃ§os.
3. Cadastro de frota e motoristas (fase 2).

## 5. Modelagem de dados

Banco local em **SQLite** no MVP, migraÃ§Ã£o posterior para **Supabase (Postgres + PostGIS)** sem mudanÃ§a de schema.

```sql
cidade(id, nome, uf, bbox_lat_lng)

posto(id, cidade_id, nome, bandeira, endereco, lat, lng, ativo)

combustivel(id, codigo)
  -- gasolina_comum, gasolina_aditivada,
  -- etanol, diesel_s10, diesel_s500, gnv

preco(id, posto_id, combustivel_id, valor, coletado_em, fonte)
  -- histÃ³rico completo, nunca sobrescreve; query pega o mais recente

veiculo_tipo(id, nome)                    -- moto, carro, caminhao_leve
veiculo_perfil(id, tipo_id, modelo, combustivel_id, km_por_litro)

usuario(id, telegram_id, nome, perfil)    -- motorista, gestor, admin
usuario_veiculo(usuario_id, veiculo_perfil_id)

consulta(id, usuario_id, lat, lng, combustivel_id, feita_em, resposta_json)
  -- log de consultas: base pra anÃ¡lise e treino de IA na fase 3
```

Ãndices principais:

- `preco(posto_id, combustivel_id, coletado_em DESC)`
- `posto(cidade_id, ativo)`
- Geoespacial em `posto(lat, lng)` â€” PostGIS quando migrar pro Supabase.

## 6. Algoritmo inicial â€” sem IA

UsuÃ¡rio na posiÃ§Ã£o `(lat0, lng0)` quer combustÃ­vel `C` com veÃ­culo de consumo `k` km/l:

```
Para cada posto p da cidade com preÃ§o atual de C:
  d = distancia(p, posicao_usuario)          # Haversine no MVP
  desvio_ida_volta = 2 * d
  litros_desvio = desvio_ida_volta / k
  custo_desvio = litros_desvio * preco_C(p)
  custo_efetivo = preco_C(p) + (custo_desvio / capacidade_tanque_usuario)
Ordena ascendente por custo_efetivo
Retorna top 3
```

Regras de negÃ³cio embutidas:

- Ignora postos com preÃ§o desatualizado alÃ©m de X dias (parametrizÃ¡vel por cidade).
- Ignora postos inativos ou fora da `bbox` da cidade.
- Se `d > raio_maximo` (ex.: 8 km moto, 15 km caminhÃ£o), descarta.
- Empate Ã© desempatado pelo preÃ§o bruto mais baixo.

Isso valida o produto **sem nenhum modelo de ML** e jÃ¡ gera economia mensurÃ¡vel. Todo log em `consulta` vira dataset pra fase de IA.

## 7. Stack recomendada

| Camada             | Escolha                                | Motivo                                               |
| ------------------ | -------------------------------------- | ---------------------------------------------------- |
| Linguagem          | Python 3.12                            | Ecossistema de dados, bot e web num stack sÃ³         |
| Backend/API        | FastAPI                                | RÃ¡pido, async, OpenAPI automÃ¡tico                    |
| Bot                | python-telegram-bot v21                | Maduro, suporta webhooks                             |
| DB MVP             | SQLite (arquivo local)                 | Zero infra, backup fÃ¡cil                             |
| DB fase 2          | Supabase (Postgres + PostGIS)          | Auth, realtime, API pronta, migraÃ§Ã£o trivial         |
| ORM                | SQLAlchemy 2 + Alembic                 | Migrations iguais em SQLite e Postgres               |
| Web frontend       | HTMX + Jinja (ou Next.js se quiser SPA)| HTMX acelera MVP e evita stack duplo                 |
| Mapa               | Leaflet + OpenStreetMap                | GrÃ¡tis, suficiente pro MVP                           |
| Roteamento         | OSRM pÃºblico ou self-hosted            | GrÃ¡tis, sem Google Maps no comeÃ§o                    |
| Deploy MVP         | Railway ou Fly.io                      | 1 container, barato                                  |
| Observabilidade    | Logfire ou Sentry free                 | Erros e traces                                       |

Tudo roda num Ãºnico processo Python no MVP. Sem microserviÃ§os.

## 8. Roadmap

- **Fase 1 â€” MVP (4â€“6 semanas):** cidade piloto, bot + web, cadastro manual.
- **Fase 2 â€” Escala operacional:** migraÃ§Ã£o pra Supabase, multi-cidade, atualizaÃ§Ã£o de preÃ§os pelo celular via bot interno, OCR (Tesseract) em foto de bomba.
- **Fase 3 â€” InteligÃªncia:** previsÃ£o de preÃ§o por posto (Prophet / LightGBM), detecÃ§Ã£o de anomalia, ranking personalizado por histÃ³rico do motorista.
- **Fase 4 â€” Frota:** dashboard de economia agregada, integraÃ§Ã£o com rastreador, relatÃ³rio fiscal, API pra TMS.
- **Fase 5 â€” Rede:** expansÃ£o nacional, freemium, parcerias com redes de postos.

## 9. Riscos e mitigaÃ§Ã£o

| Risco                                            | MitigaÃ§Ã£o                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------- |
| PreÃ§os desatualizados matam a confianÃ§a          | Mostrar timestamp do preÃ§o no bot; alerta diÃ¡rio pro operador             |
| Operador cansa de atualizar                      | Bot interno `/atualizar_preco` pra operador fazer do celular em 10s       |
| Cidade piloto pequena demais pra provar ROI      | Escolher cidade com volume real de entregas                               |
| Motorista nÃ£o adota bot                          | Onboarding via grupo de WhatsApp da frota e vÃ­deo curto                   |
| PrecisÃ£o de consumo (km/l) Ã© chute               | ComeÃ§ar com mÃ©dias da categoria e refinar com histÃ³rico do prÃ³prio motorista |

## 10. Plano de execuÃ§Ã£o

| Semana | Entrega                                                                     |
| ------ | --------------------------------------------------------------------------- |
| 1      | Schema SQLite, seed de postos/veÃ­culos da cidade piloto, importaÃ§Ã£o CSV     |
| 2      | Core do cÃ¡lculo de custo efetivo e testes; endpoint FastAPI `/melhor-posto` |
| 3      | Bot Telegram: `/start`, `/melhor`, `/postos`, recebimento de localizaÃ§Ã£o    |
| 4      | Web admin (HTMX) pra atualizar preÃ§o e mapa Leaflet pÃºblico                 |
| 5      | Deploy, piloto com 3â€“5 motoristas reais, coleta de feedback                 |
| 6      | Ajustes, mÃ©tricas de economia real, decisÃ£o de migrar pra Supabase          |

---

## DecisÃµes pendentes

Dois pontos ainda precisam ser definidos antes da estrutura de cÃ³digo ser criada:

1. **Cidade piloto** â€” qual cidade vai rodar o MVP? Idealmente onde jÃ¡ exista contato com uma frota disposta a testar.
2. **OperaÃ§Ã£o dos preÃ§os** â€” quem atualiza os preÃ§os no dia a dia: o prÃ³prio mantenedor, um operador contratado, ou a frota parceira mandando foto da bomba pelo bot?

Uma vez definidos, o prÃ³ximo passo Ã© criar o scaffold Python (FastAPI + bot + SQLite + Alembic) e o primeiro seed de dados.

---

Projeto adaptado da comunidade INEMA TDS - Marca Arckhe