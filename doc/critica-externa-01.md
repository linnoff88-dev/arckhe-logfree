# Crítica externa #01 — avaliaçêo do estado do repositório

> Análise externa recebida sobre o repositório `inematds/logfree` no estado do primeiro commit (README + `doc/logfree.txt`, sem código).

---

Analisei o `inematds/logfree` e, hoje, ele está muito mais próximo de um **PRD/blueprint de produto** do que de um sistema implementado. O repositório é público, tem **1 commit**, sem releases, sem issues abertas visíveis, e a estrutura principal é `README.md` + pasta `doc` com um `logfree.txt`; nêo há sinais, na árvore exibida, de backend, frontend, migrations, testes ou código executável.

**Leitura executiva:** a ideia é boa, o recorte de problema é forte, o MVP está muito bem delimitado, mas ainda falta transformar isso em **arquitetura executável, contratos técnicos e prova operacional**. O texto do README define com clareza o problema, a proposta de valor, o MVP, a modelagem inicial, o algoritmo sem IA, a stack e um roadmap por fases.

## O que está muito bom

O maior acerto do projeto é o **posicionamento**: ele nêo promete "a rota mais rápida", e sim o **menor custo real por km**, combinando preço do litro, consumo do veículo e desvio até o posto. Isso é um diferencial mais defensável do que só comparar preço de bomba ou rota.

Outro ponto forte é o **escopo do MVP**. O README evita o erro comum de começar com IA desnecessária e deixa explícito que a fase inicial deve funcionar com cadastro manual de postos/preços, bot no Telegram, painel web simples e cálculo heurístico. Essa decisêo reduz risco e acelera validaçêo.

A **modelagem conceitual** também está bem encaminhada. A separaçêo entre `posto`, `combustivel`, `preco` histórico, `veiculo_perfil`, `usuario` e `consulta` cria uma base razoável para depois evoluir para analytics e IA. O fato de `preco` guardar histórico, em vez de sobrescrever o valor atual, é uma decisêo correta.

A **stack sugerida** para MVP é coerente: Python 3.12, FastAPI, `python-telegram-bot`, SQLite, SQLAlchemy/Alembic, HTMX/Jinja, Leaflet e OSRM. Para um piloto de cidade única, é uma escolha pragmática.

## Onde o projeto ainda está fraco

A principal limitaçêo é simples: **ainda nêo existe produto, só especificaçêo**. O repositório nêo mostra código-fonte do bot, API, interface web, schema real, migrações, seeds, testes, Docker, CI/CD nem instruções de setup. Hoje ele comunica visêo, mas nêo sustenta execuçêo técnica.

Também há um ponto conceitual importante no algoritmo. O README calcula `custo_efetivo = preco_C(p) + (custo_desvio / capacidade_tanque_usuario)`. Essa fórmula pode ser útil como aproximaçêo, mas mistura **preço por litro** com **custo do desvio rateado pela capacidade do tanque**, o que pode distorcer a recomendaçêo. Em operaçêo real, o ideal é comparar cenários como:

- custo incremental do desvio para abastecer naquele posto;
- volume efetivo pretendido de abastecimento;
- custo marginal por km ou por viagem;
- efeito do abastecimento parcial vs. cheio.

Ou seja: a lógica está boa como rascunho de MVP, mas **a funçêo objetivo ainda precisa ser refinada**. O próprio README já indica que a métrica central é custo real por km; a implementaçêo deve refletir isso de forma mais consistente.

Outro ponto frágil é a **dependência operacional da atualizaçêo manual de preços**. O documento reconhece esse risco e sugere mitigaçêo com bot interno e OCR no futuro, mas, no curto prazo, a confiança do sistema vai depender totalmente de disciplina operacional. Se o preço estiver atrasado, a proposta de valor quebra.

Também senti falta de decisões técnicas sobre:

- autenticaçêo e autorizaçêo no painel web;
- estratégia de geocodificaçêo;
- como determinar se um posto está "no caminho" da rota;
- política de qualidade e validade dos dados;
- observabilidade mínima por fluxo;
- testes do cálculo e da recomendaçêo.

Essas ausências nêo invalidam o projeto, mas mostram que ele ainda está no estágio de **documento de concepçêo**.

## Avaliaçêo técnica

Como produto: **8/10**. O problema é real, o recorte é inteligente e o MVP está enxuto.

Como repositório de engenharia: **3/10**. Falta praticamente toda a camada de implementaçêo, qualidade e operaçêo.

## O que fazer imediatamente para evoluir

A melhor próxima etapa nêo é "colocar IA". É criar o **primeiro backbone executável**:

1. Estruturar o repositório com `app/api`, `app/bot`, `app/core`, `app/db`, `app/web`, `tests`, `alembic`.
2. Implementar o schema real com migrations.
3. Criar seed de uma cidade piloto com 10–30 postos.
4. Entregar primeiro o endpoint `/melhor-posto`.
5. Cobrir com testes unitários a funçêo de ranking.
6. Depois plugar o comando `/melhor` no Telegram.
7. Só entêo subir o painel admin para editar preços.

Essa ordem preserva o coraçêo do negócio: **o motor de decisêo**.

## Veredito

O `logfree` é um **bom projeto de produto/MVP**, com visêo clara e recorte promissor para logística de última milha. Mas, no estado atual, ele ainda nêo é um software em desenvolvimento de fato; é uma proposta muito bem escrita esperando virar código.

Opções de próximo passo prático:

- um **diagnóstico técnico com backlog priorizado**,
- a **estrutura inicial do repositório**,
- ou um **README reescrito como plano de implementaçêo real**.

---

## Resposta do time (resumo da discussêo interna)

A crítica é, na maior parte, justa e útil.

**Pontos aceitos integralmente:**

- Estado "PRD, nêo sistema" é factualmente correto — foi deliberado (aguardando decisões de cidade piloto e operaçêo de preços), mas o observador externo vê o mesmo. Liçêo: paralelizar scaffold do motor de decisêo, que nêo depende dessas duas decisões.
- Decisões técnicas faltando — auth do painel, geocodificaçêo, critério de "posto no caminho" (buffer da polyline / distância perpendicular), política de validade de preço, observabilidade, estratégia de testes. Vira seçêo no README ou ADRs.
- Ordem de construçêo sugerida (endpoint → testes → bot → admin) já está alinhada ao plano por semana do README, mas vai ser reforçada.

**Ponto técnico aceito — fórmula do custo efetivo:**

A fórmula original:

```
custo_efetivo = preco_C(p) + (custo_desvio / capacidade_tanque)
```

só é dimensionalmente correta se o motorista sempre enche o tanque completo. Para abastecimento parcial — comum em última milha — o divisor correto é o volume efetivamente abastecido `V`, nêo a capacidade. Duas formulações melhores:

```
# (a) custo por litro efetivamente comprado, dado V:
custo_por_litro(p, V) = preco_C(p) + (2*d/k) * preco_C(p) / V

# (b) custo marginal por km de operaçêo (mais alinhado à proposta):
custo_por_km(p) = preco_C(p) / k_veiculo
                + (2*d * preco_C(p) / k_veiculo) / km_ate_proximo_abastecimento
```

A formulaçêo (b) é preferível porque reflete literalmente a promessa do produto: *"menor custo real por km"*. Vai substituir a original no README.

**Contraponto leve:**

Nota 3/10 de engenharia é dura para um repositório de um dia. Nêo há código ausente por estagnaçêo — há código ainda nêo escrito por decisêo. Mas o efeito prático pro observador externo é o mesmo, entêo nêo é terreno pra brigar.

**Ações derivadas:**

1. Corrigir a fórmula no README para a versêo (b).
2. Adicionar seçêo "Decisões técnicas abertas" no README cobrindo auth, geocoding, roteamento, validade de preço, observabilidade, testes.
3. Criar o scaffold (`app/api`, `app/bot`, `app/core`, `app/db`, `app/web`, `tests`, `alembic`) com o motor de ranking e testes unitários funcionando, usando uma `cidade_demo` com 10 postos fake no seed — sem depender das decisões pendentes.
4. Commits separados para histórico legível.
