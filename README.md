ï»¿# LogFree

SoluêÂ§êÂ£o local por cidade que indica, para frotas de êÂºltima milha (moto, carro, caminhêÂ£o leve), o posto de combustêÂ­vel com **melhor custo real por km rodado** ê¢â¬â nêÂ£o apenas o preêÂ§o mais baixo na bomba.

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

Pule o `python3 -m venv .venv` se jêÂ¡ tiver `.venv` no projeto.

### 2. DB + seed

```bash
alembic upgrade head           # cria tabelas + triggers
python -m app.db.seed          # popula cidade_demo (15 postos, 6 combustêÂ­veis)
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

Abra <http://localhost:8000/admin/login> no browser, entre com `admin@local` + a senha. VocêÂª vêÂª as cidades, lista de postos com formulêÂ¡rio HTMX para registrar preêÂ§o, mêÂ©tricas. Login errado, CSRF ausente e cross-city sêÂ£o bloqueados (testes cobrem).

### 6. Testes

```bash
pytest -q                                   # todos (~98)
pytest tests/unit -q                        # sêÂ³ unitêÂ¡rios (rêÂ¡pido)
pytest tests/integration -q                 # API + admin + bot
pytest --cov=app --cov-report=term-missing  # com cobertura
ruff check app tests                        # lint
```

### 7. CLI êÂºteis

```bash
logfree audit-verify                                            # checa cadeia hash do evento_audit
logfree audit-unblock --approver=admin@x --motivo="..."         # break-glass (precisa AUDIT_UNBLOCK_KEY)
logfree flag-set --chave=ranking.osrm_enabled --valor=true
logfree flag-set --chave=kill_switch.global --valor=true        # desliga /melhor-posto (503)
logfree replay-tombstones                                       # pêÂ³s-restore: re-anonimiza usuêÂ¡rios tombstoned
```

### 8. Bot Telegram (opcional, sêÂ³ com token real)

NêÂ£o funciona sem token real. Se quiser testar:

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

Sem fazer isso, o webhook responde 503. Os fluxos do bot estêÂ£o cobertos por testes mockando o JSON do Telegram.

---

### Endpoints principais

- `POST /melhor-posto` ê¢â¬â auth obrigatêÂ³ria (sessêÂ£o admin OU `X-Bot-Token` + `X-Bot-User-Telegram-Id`).
- `POST /bot/webhook/{path_secret}` ê¢â¬â webhook Telegram (header `X-Telegram-Bot-Api-Secret-Token` validado).
- `GET /admin/login`, `GET /admin/`, `GET /admin/postos/{cidade_id}` ê¢â¬â UI HTMX.
- `GET /internal/ready`, `GET /internal/metrics` ê¢â¬â healthcheck e mêÂ©tricas.

### Erros comuns

| Sintoma | Causa | SoluêÂ§êÂ£o |
|---|---|---|
| `no such table: alembic_version` | migration nêÂ£o rodou | `alembic upgrade head` |
| `combustivel desconhecido` na API | falta seed da 2a migration | `alembic upgrade head` (gera os 6 combustêÂ­veis) |
| `flock ocupado` ao iniciar uvicorn | outro processo segurando o owner lease | `pkill -f uvicorn` (ou cheque o PID que estêÂ¡ com o lock) |
| `401` em `/melhor-posto` com bot token | `X-Bot-Token` errado, `X-Bot-User-Telegram-Id` inexistente, ou usuêÂ¡rio sem `opt_in_lgpd=1` | Confira `.env` e `usuario.opt_in_lgpd`; o motorista demo do seed (`telegram_id=100001`) jêÂ¡ vem com opt-in. |
| `403` em `/melhor-posto` | sem opt-in LGPD | bot deve mandar `/aceitar_termos` antes de `/melhor` |

VariêÂ¡veis de ambiente em `.env.example`. Em produêÂ§êÂ£o, configure pela plataforma (Fly secrets / Railway env).

Plano completo: ver [`PLAN.md`](PLAN.md) (3 rodadas de revisêÂ£o adversarial ê¢â¬â engenharia, seguranêÂ§a, ops/SRE).

## Runbooks

[`runbook/README.md`](runbook/README.md) lista todos os runbooks (rollback, restore, audit chain break, OSRM degradaêÂ§êÂ£o, segredos, capacidade, etc).

---

## 1. Problema

Motoristas de êÂºltima milha perdem margem abastecendo no posto errado. A decisêÂ£o êÂ© tomada pelo preêÂ§o da placa, mas o que importa êÂ© o **custo efetivo por km**, que depende de:

- preêÂ§o do litro
- consumo do veêÂ­culo naquele combustêÂ­vel
- desvio que o motorista faz pra chegar atêÂ© o posto

Hoje ninguêÂ©m entrega isso. Waze e Google mostram rota. Apps de preêÂ§o mostram preêÂ§o. Nenhum junta os dois com o perfil do veêÂ­culo e o contexto de entrega urbana.

## 2. Proposta de valor

> "O posto mais barato **pra vocêÂª, hoje, considerando seu veêÂ­culo e onde vocêÂª estêÂ¡.**"

Diferenciais:

- Curadoria **local por cidade** ê¢â¬â dados confiêÂ¡veis, sem crowdsourcing caêÂ³tico.
- CêÂ¡lculo de **custo efetivo por km** por veêÂ­culo, nêÂ£o preêÂ§o bruto.
- Acesso por **Telegram**, onde o motorista jêÂ¡ vive, e **web** pro gestor da frota.
- Pensado pra **êÂºltima milha**: raio curto, muitas paradas, decisêÂ£o rêÂ¡pida.

## 3. Escopo do MVP

**Entra:**

- Cadastro manual de postos e preêÂ§os (operador da cidade piloto).
- Cadastro de veêÂ­culos (tipo, modelo, combustêÂ­vel, km/l).
- Bot Telegram respondendo:
  - `/postos` ê¢â¬â lista os N postos mais baratos por combustêÂ­vel na cidade.
  - `/melhor <combustêÂ­vel>` ê¢â¬â melhor posto pro veêÂ­culo do usuêÂ¡rio, considerando localizaêÂ§êÂ£o enviada.
  - `/rota origem ê¢â â destino` ê¢â¬â rota sugerida + posto recomendado no caminho.
- Web: mapa da cidade com postos, tabela ordenêÂ¡vel, painel admin pra atualizar preêÂ§os.
- RelatêÂ³rio simples por motorista/frota (economia estimada).

**Fica pra depois:** IA, previsêÂ£o de preêÂ§os, integraêÂ§êÂ£o com cartêÂ£o-frota, OBD-II, multi-cidade, app nativo.

## 4. Fluxos principais

**Motorista (Telegram):**

1. `/start` ê¢â¬â cadastra veêÂ­culo (tipo + km/l mêÂ©dio).
2. Envia localizaêÂ§êÂ£o.
3. `/melhor diesel` ê¢â¬â bot consulta DB, calcula custo efetivo, devolve top 3 postos com preêÂ§o, distêÂ¢ncia, desvio e economia estimada vs. o mais prêÂ³ximo.

**Gestor (Web):**

1. Login admin, tela de postos, edita preêÂ§o do dia.
2. Dashboard: mapa, lista, histêÂ³rico de preêÂ§os.
3. Cadastro de frota e motoristas (fase 2).

## 5. Modelagem de dados

Banco local em **SQLite** no MVP, migraêÂ§êÂ£o posterior para **Supabase (Postgres + PostGIS)** sem mudanêÂ§a de schema.

```sql
cidade(id, nome, uf, bbox_lat_lng)

posto(id, cidade_id, nome, bandeira, endereco, lat, lng, ativo)

combustivel(id, codigo)
  -- gasolina_comum, gasolina_aditivada,
  -- etanol, diesel_s10, diesel_s500, gnv

preco(id, posto_id, combustivel_id, valor, coletado_em, fonte)
  -- histêÂ³rico completo, nunca sobrescreve; query pega o mais recente

veiculo_tipo(id, nome)                    -- moto, carro, caminhao_leve
veiculo_perfil(id, tipo_id, modelo, combustivel_id, km_por_litro)

usuario(id, telegram_id, nome, perfil)    -- motorista, gestor, admin
usuario_veiculo(usuario_id, veiculo_perfil_id)

consulta(id, usuario_id, lat, lng, combustivel_id, feita_em, resposta_json)
  -- log de consultas: base pra anêÂ¡lise e treino de IA na fase 3
```

êÂndices principais:

- `preco(posto_id, combustivel_id, coletado_em DESC)`
- `posto(cidade_id, ativo)`
- Geoespacial em `posto(lat, lng)` ê¢â¬â PostGIS quando migrar pro Supabase.

## 6. Algoritmo inicial ê¢â¬â sem IA

UsuêÂ¡rio na posiêÂ§êÂ£o `(lat0, lng0)` quer combustêÂ­vel `C` com veêÂ­culo de consumo `k` km/l:

```
Para cada posto p da cidade com preêÂ§o atual de C:
  d = distancia(p, posicao_usuario)          # Haversine no MVP
  desvio_ida_volta = 2 * d
  litros_desvio = desvio_ida_volta / k
  custo_desvio = litros_desvio * preco_C(p)
  custo_efetivo = preco_C(p) + (custo_desvio / capacidade_tanque_usuario)
Ordena ascendente por custo_efetivo
Retorna top 3
```

Regras de negêÂ³cio embutidas:

- Ignora postos com preêÂ§o desatualizado alêÂ©m de X dias (parametrizêÂ¡vel por cidade).
- Ignora postos inativos ou fora da `bbox` da cidade.
- Se `d > raio_maximo` (ex.: 8 km moto, 15 km caminhêÂ£o), descarta.
- Empate êÂ© desempatado pelo preêÂ§o bruto mais baixo.

Isso valida o produto **sem nenhum modelo de ML** e jêÂ¡ gera economia mensurêÂ¡vel. Todo log em `consulta` vira dataset pra fase de IA.

## 7. Stack recomendada

| Camada             | Escolha                                | Motivo                                               |
| ------------------ | -------------------------------------- | ---------------------------------------------------- |
| Linguagem          | Python 3.12                            | Ecossistema de dados, bot e web num stack sêÂ³         |
| Backend/API        | FastAPI                                | RêÂ¡pido, async, OpenAPI automêÂ¡tico                    |
| Bot                | python-telegram-bot v21                | Maduro, suporta webhooks                             |
| DB MVP             | SQLite (arquivo local)                 | Zero infra, backup fêÂ¡cil                             |
| DB fase 2          | Supabase (Postgres + PostGIS)          | Auth, realtime, API pronta, migraêÂ§êÂ£o trivial         |
| ORM                | SQLAlchemy 2 + Alembic                 | Migrations iguais em SQLite e Postgres               |
| Web frontend       | HTMX + Jinja (ou Next.js se quiser SPA)| HTMX acelera MVP e evita stack duplo                 |
| Mapa               | Leaflet + OpenStreetMap                | GrêÂ¡tis, suficiente pro MVP                           |
| Roteamento         | OSRM pêÂºblico ou self-hosted            | GrêÂ¡tis, sem Google Maps no comeêÂ§o                    |
| Deploy MVP         | Railway ou Fly.io                      | 1 container, barato                                  |
| Observabilidade    | Logfire ou Sentry free                 | Erros e traces                                       |

Tudo roda num êÂºnico processo Python no MVP. Sem microserviêÂ§os.

## 8. Roadmap

- **Fase 1 ê¢â¬â MVP (4ê¢â¬â6 semanas):** cidade piloto, bot + web, cadastro manual.
- **Fase 2 ê¢â¬â Escala operacional:** migraêÂ§êÂ£o pra Supabase, multi-cidade, atualizaêÂ§êÂ£o de preêÂ§os pelo celular via bot interno, OCR (Tesseract) em foto de bomba.
- **Fase 3 ê¢â¬â InteligêÂªncia:** previsêÂ£o de preêÂ§o por posto (Prophet / LightGBM), detecêÂ§êÂ£o de anomalia, ranking personalizado por histêÂ³rico do motorista.
- **Fase 4 ê¢â¬â Frota:** dashboard de economia agregada, integraêÂ§êÂ£o com rastreador, relatêÂ³rio fiscal, API pra TMS.
- **Fase 5 ê¢â¬â Rede:** expansêÂ£o nacional, freemium, parcerias com redes de postos.

## 9. Riscos e mitigaêÂ§êÂ£o

| Risco                                            | MitigaêÂ§êÂ£o                                                                 |
| ------------------------------------------------ | ------------------------------------------------------------------------- |
| PreêÂ§os desatualizados matam a confianêÂ§a          | Mostrar timestamp do preêÂ§o no bot; alerta diêÂ¡rio pro operador             |
| Operador cansa de atualizar                      | Bot interno `/atualizar_preco` pra operador fazer do celular em 10s       |
| Cidade piloto pequena demais pra provar ROI      | Escolher cidade com volume real de entregas                               |
| Motorista nêÂ£o adota bot                          | Onboarding via grupo de WhatsApp da frota e vêÂ­deo curto                   |
| PrecisêÂ£o de consumo (km/l) êÂ© chute               | ComeêÂ§ar com mêÂ©dias da categoria e refinar com histêÂ³rico do prêÂ³prio motorista |

## 10. Plano de execuêÂ§êÂ£o

| Semana | Entrega                                                                     |
| ------ | --------------------------------------------------------------------------- |
| 1      | Schema SQLite, seed de postos/veêÂ­culos da cidade piloto, importaêÂ§êÂ£o CSV     |
| 2      | Core do cêÂ¡lculo de custo efetivo e testes; endpoint FastAPI `/melhor-posto` |
| 3      | Bot Telegram: `/start`, `/melhor`, `/postos`, recebimento de localizaêÂ§êÂ£o    |
| 4      | Web admin (HTMX) pra atualizar preêÂ§o e mapa Leaflet pêÂºblico                 |
| 5      | Deploy, piloto com 3ê¢â¬â5 motoristas reais, coleta de feedback                 |
| 6      | Ajustes, mêÂ©tricas de economia real, decisêÂ£o de migrar pra Supabase          |

---

## DecisêÂµes pendentes

Dois pontos ainda precisam ser definidos antes da estrutura de cêÂ³digo ser criada:

1. **Cidade piloto** ê¢â¬â qual cidade vai rodar o MVP? Idealmente onde jêÂ¡ exista contato com uma frota disposta a testar.
2. **OperaêÂ§êÂ£o dos preêÂ§os** ê¢â¬â quem atualiza os preêÂ§os no dia a dia: o prêÂ³prio mantenedor, um operador contratado, ou a frota parceira mandando foto da bomba pelo bot?

Uma vez definidos, o prêÂ³ximo passo êÂ© criar o scaffold Python (FastAPI + bot + SQLite + Alembic) e o primeiro seed de dados.

---

Projeto  - ARCKHÉe