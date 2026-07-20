# Códigos dos slides — explicação

Este arquivo acompanha a apresentação (`index.html`). Cada seção corresponde a um
slide e explica **o que o código faz, por que está escrito daquele jeito e o que
responder se perguntarem**. O código dos slides é uma versão enxuta do que está
no projeto — os arquivos completos ficam em `ModelTrace-Pipeline/`.

---

## Slide 02 — Coleções principais

```python
MAIN_COLLECTIONS = [
    "organizations", "users", "projects",
    "models", "model_versions",
    "predictions",
    "metrics_snapshots", "entity_notes", "audit_events",
]

for nome in MAIN_COLLECTIONS:
    db.create_collection(nome)

db.predictions.create_index([("project_id", 1), ("score", -1)])
```

**O que faz:** cria as nove coleções e um índice composto em `predictions`.

**Por que assim:**

- A hierarquia é `organization → project → model → model_version → prediction`.
  Cada nível é uma coleção separada porque são consultados de forma independente
  (a tela de projetos não precisa carregar predições).
- `predictions` é a coleção central: é onde os dados crescem e onde quase toda
  consulta bate.
- O índice é `(project_id, score desc)` porque essa é literalmente a consulta da
  tela: *“os casos mais críticos deste projeto”*. A ordem dos campos importa —
  primeiro o campo de igualdade (`project_id`), depois o de ordenação (`score`).

**Documento de predição:** a explicação do modelo (`explanations`) fica
**embutida** no documento, não em outra coleção. Motivo: ela nunca é lida sem a
predição. É o caso clássico de embedding em vez de referência — evita um `lookup`
em toda leitura.

**Se perguntarem “por que não normalizar tudo?”:** porque em documento a regra é
modelar pelo padrão de leitura. O que é lido junto, fica junto.

---

## Slide 04 a 07 — CRUD

### INSERT

```python
def insert_prediction(self, doc):
    self.db.predictions.insert_one(doc)
    self.redis_ingest(doc)
    return doc
```

Grava no MongoDB (a fonte da verdade) e atualiza os contadores do Redis na mesma
operação. O painel não recalcula nada depois: os números já sobem prontos.

### FIND

```python
def find_predictions(self, filtro, sort=None, limit=20):
    cursor = self.db.predictions.find(filtro)
    if sort:
        cursor = cursor.sort(sort)
    return list(cursor.limit(limit))

find_predictions({"project_id": "proj::evasao", "score": {"$gte": 0.85}},
                 sort=[("score", -1)])
```

O `find` devolve um **cursor** — nada é lido do banco até iterar. Por isso o
`sort` e o `limit` são aplicados no cursor: o MongoDB resolve tudo no servidor e
só manda os 20 documentos. É essa consulta que o índice do slide 02 atende.

### UPDATE

```python
def update_feedback(self, pred_id, valor_observado):
    pred = self.db.predictions.find_one({"_id": pred_id})
    acertou = (pred["score"] >= 0.5) == bool(valor_observado)

    self.db.predictions.update_one(
        {"_id": pred_id},
        {"$set": {"observed_value": valor_observado,
                  "outcome": "TP" if acertou else "FP"}})
```

O analista informa o que aconteceu de verdade. O código compara com o que o
modelo previu e classifica em TP (acertou o positivo) ou FP (falso alarme).

**Por que isso importa:** sem esse campo não existe métrica de qualidade. É ele
que a Aggregation 1 usa para montar o placar dos modelos.

### DELETE

```python
def delete_prediction(self, pred_id):
    r = self.db.predictions.delete_one({"_id": pred_id})
    self.append_audit("delete", {"pred_id": pred_id})
    return r.deleted_count == 1
```

Remove e registra o evento em `audit_events`. `deleted_count` confirma se algo
foi de fato apagado (o filtro pode não ter casado com nada).

---

## Slide 08 — Aggregation 1: placar dos modelos

```javascript
db.predictions.aggregate([
  {"$match":  {"outcome": {"$ne": "PENDING"}}},
  {"$group":  {"_id": "$version_id",
               "n":  {"$sum": 1},
               "tp": {"$sum": {"$cond": [{"$eq": ["$outcome", "TP"]}, 1, 0]}}}},
  {"$set":    {"precisao": {"$divide": ["$tp", "$n"]}}},
  {"$lookup": {"from": "model_versions", "localField": "_id",
               "foreignField": "_id", "as": "v"}},
  {"$unwind": "$v"},
  {"$project":{"modelo": "$v.label", "n": 1, "precisao": 1}},
  {"$sort":   {"precisao": -1}},
  {"$merge":  {"into": "metrics_snapshots"}}
])
```

**Pergunta que responde:** qual versão de modelo está acertando mais.

Etapa por etapa:

| Etapa | O que faz | Por que nessa posição |
|---|---|---|
| `$match` | descarta predições ainda sem resposta | **primeiro**, para reduzir o volume antes de agrupar |
| `$group` | junta por versão, conta o total e os acertos | o `$cond` conta condicionalmente: soma 1 só quando o outcome é TP |
| `$set` | calcula a precisão (`tp / n`) | depois do group, quando os totais já existem |
| `$lookup` | busca o nome da versão em outra coleção | é o "join" do MongoDB |
| `$unwind` | o `$lookup` devolve um array; isso vira um objeto | sem ele, o campo seria `[{...}]` |
| `$project` | escolhe só os campos que interessam | reduz o que trafega |
| `$sort` | melhor precisão primeiro | — |
| `$merge` | grava o resultado numa coleção | o painel lê o resultado pronto |

**Pontos que costumam ser perguntados:**

- *Por que `$match` primeiro?* Porque a ordem dos estágios é a ordem de execução.
  Filtrar antes de agrupar significa agrupar menos documentos.
- *`$set` ou `$addFields`?* São a mesma coisa; `$set` é o nome moderno.
- *Por que `$merge` e não só devolver?* Porque o resultado vira uma coleção
  consultável (`metrics_snapshots`), sem recalcular a cada abertura da tela.

---

## Slide 09 — Aggregation 2: o que mais pesa no risco

```javascript
db.predictions.aggregate([
  {"$match":  {"project_id": projeto}},
  {"$unwind": "$explanations"},
  {"$group":  {"_id": "$explanations.feature",
               "impacto": {"$avg": "$explanations.impact"},
               "casos":   {"$sum": 1}}},
  {"$set":    {"impacto": {"$round": ["$impacto", 3]}}},
  {"$sort":   {"impacto": -1}},
  {"$limit":  10}
])
```

**Pergunta que responde:** quais fatores mais empurram os casos para risco alto.

O estágio central é o `$unwind`. Cada predição carrega um **array** de
explicações; o `$unwind` transforma um documento com 3 explicações em 3
documentos, um por fator. Só depois disso dá para agrupar **por fator** e tirar a
média do impacto.

O `$sample` (também citado no slide) sorteia documentos aleatórios — serve para
inspeção manual, quando se quer olhar alguns casos reais sem viés de ordenação.
O `$facet` permite rodar vários sub-pipelines na mesma passada (por exemplo, o
ranking e uma contagem geral ao mesmo tempo).

**A frase importante do slide:** essa agregação é **plana**. Ela diz *quais*
fatores pesam, mas não diz *quais casos se parecem entre si*. Para isso seria
preciso comparar cada caso com todos os outros — que é exatamente o que o grafo
faz no slide 14. Esse contraste é a justificativa de existir um banco de grafos
no projeto.

---

## Slide 11 — Redis: estruturas comuns

```python
r.hincrby("mt:counters:evasao", "predictions", 1)      # Hash

r.zadd("mt:risk", {"aluno#014": 0.91})                 # Sorted Set
r.zrevrange("mt:risk", 0, 4, withscores=True)

r.lpush("mt:feed", evento)                             # List
r.ltrim("mt:feed", 0, 49)

r.setex("mt:cache:top", 60, json.dumps(dados))         # String + TTL

usos = r.incr("mt:rate:chave")                         # limite de uso
if usos == 1: r.expire("mt:rate:chave", 60)
permitido = usos <= 5
```

| Estrutura | Uso no sistema | Por que essa e não outra |
|---|---|---|
| Hash | contadores por modelo | agrupa vários campos sob uma chave; incremento atômico |
| Sorted Set | ranking de risco | mantém ordenado por score na escrita; ler o top-5 é imediato |
| List | feed dos últimos casos | `lpush` + `ltrim` = fila de tamanho fixo, sem limpeza manual |
| String + TTL | cache de consulta cara | o `setex` expira sozinho; não existe cache velho |
| Contador + expire | limite de chamadas | a janela morre junto com a chave |

**Detalhe do rate limit:** o `expire` só é chamado quando o contador vale 1, ou
seja, na primeira chamada da janela. Se fosse chamado sempre, a janela nunca
fecharia — ela seria renovada a cada requisição.

**Prefixo `mt:`** — todas as chaves usam namespace para não colidir com outras
aplicações no mesmo Redis.

---

## Slide 12 — Redis: estruturas probabilísticas

```python
r.execute_command("BF.RESERVE", "mt:vistos", 0.01, 100000)   # Bloom Filter
r.execute_command("BF.ADD", "mt:vistos", "pred::0007")
r.execute_command("BF.EXISTS", "mt:vistos", "pred::0007")

r.pfadd("mt:entidades", "aluno#014")                          # HyperLogLog
r.pfcount("mt:entidades")

r.execute_command("CMS.INCRBY", "mt:fatores", "faltas", 1)    # Count-Min Sketch
r.execute_command("CMS.QUERY", "mt:fatores", "faltas")

r.execute_command("TOPK.ADD", "mt:top", "faltas")             # Top-K
r.execute_command("TOPK.LIST", "mt:top")
```

Todas trocam **precisão exata por memória constante**.

| Estrutura | Responde | Erro que aceita |
|---|---|---|
| Bloom Filter | "já vi este caso?" | pode dizer que viu algo que não viu (falso positivo); nunca o contrário |
| HyperLogLog | "quantos distintos?" | ~1% de erro, usando alguns KB |
| Count-Min Sketch | "com que frequência?" | pode superestimar, nunca subestimar |
| Top-K | "quais os mais frequentes?" | pode errar a ordem na cauda |

**A justificativa de uso:** contar entidades distintas de forma exata exige
guardar todos os ids — memória proporcional ao volume. O HyperLogLog responde a
mesma pergunta com poucos KB. Como a decisão que depende disso ("o volume está
crescendo?") não muda por causa de 1% de erro, a troca compensa.

**Sobre o Bloom Filter:** o parâmetro `0.01` é a taxa de erro desejada e `100000`
a capacidade esperada. A garantia é assimétrica — se ele diz "não vi", é
definitivo; se diz "vi", pode ser engano. Por isso serve para *evitar
reprocessamento*, não para decisões críticas.

**Requer Redis Stack** (não o Redis básico), que traz os módulos `RedisBloom`.

---

## Slide 13 — Grafo: modelagem no Neo4j

```cypher
UNWIND $linhas AS linha
MERGE (e:Entity {id: linha.id})
SET   e.label = linha.label

UNWIND $linhas AS linha
MATCH (e:Entity  {id: linha.entity_id}),
      (f:Feature {id: linha.feature_id})
MERGE (e)-[r:RISK_FACTOR]->(f)
SET   r.weight = linha.impacto
```

**`UNWIND`** recebe uma lista inteira de linhas e processa tudo em uma única
consulta, em vez de uma consulta por nó. É a forma de carregar em lote.

**`MERGE` em vez de `CREATE`**: cria só se ainda não existir. Torna o script
repetível — rodar duas vezes não duplica nós.

**A aresta que importa:** `Entity -[:RISK_FACTOR]-> Feature`. Com ela o grafo
fica **bipartido** (casos de um lado, fatores do outro) e dois casos passam a se
conectar indiretamente pelos fatores que compartilham. É essa estrutura que o
algoritmo de similaridade consome.

---

## Slide 14 — GDS: os três algoritmos

```cypher
CALL gds.nodeSimilarity.write('grafo', {
  writeRelationshipType: 'SIMILAR_TO', writeProperty: 'score', topK: 5 })

CALL gds.louvain.write('grafo', { writeProperty: 'community_id' })

CALL gds.pageRank.write('grafo', { writeProperty: 'pagerank' })
```

| Algoritmo | O que calcula | Resultado no projeto |
|---|---|---|
| `nodeSimilarity` | semelhança entre casos pelos fatores em comum (Jaccard) | 1.155 pares |
| `louvain` | comunidades — grupos mais conectados entre si | 7 comunidades, modularidade 0.85 |
| `pageRank` | influência estrutural de cada nó | convergiu em 16 iterações |

**Por que `.write`:** os três gravam o resultado **de volta no grafo** — uma nova
aresta `SIMILAR_TO` ou uma propriedade no nó. Depois disso, a resposta vira uma
consulta Cypher comum:

```cypher
MATCH (a:Entity {id: $caso})-[s:SIMILAR_TO]->(b:Entity)
RETURN b.label, s.score ORDER BY s.score DESC LIMIT 5
```

**Sobre `topK: 5`:** guarda só os 5 vizinhos mais parecidos de cada caso. Sem
isso, o algoritmo escreveria toda combinação acima do corte e o grafo explodiria.

**Sobre a modularidade 0.85:** mede o quanto a divisão em comunidades é boa
(0 = aleatória, 1 = perfeita). 0.85 indica grupos bem definidos, ou seja, os
padrões de risco realmente se separam.

**Fluxo dos três:** projetar o grafo em memória → rodar → gravar → descartar a
projeção. A projeção é uma cópia otimizada; ela é liberada depois de cada
algoritmo para não ocupar memória.

---

## Slide 15 — Visualização do grafo

```python
from constelario import Graph, Theme

def construir(payload):
    g = Graph(title="ModelTrace", theme=Theme.relicario())

    for nome, estilo in TIPOS.items():
        g.add_type(nome, **estilo)

    for no in payload["nodes"]:
        g.add_node(no["id"], no["label"], no["type"],
                   props=no["props"],
                   community=no["props"].get("community_id"))

    for a in payload["links"]:
        g.add_edge(a["source"], a["target"], type=a["type"])

    g.set_edge_weight("score")
    g.inspector_ranking("SIMILAR_TO", title="Casos mais similares")
    return g

construir(payload).save("grafo.html")
```

**O que faz:** converte o grafo exportado do Neo4j numa página HTML interativa.

- `add_type` define a aparência de cada tipo de nó (cor, ícone, posição no
  layout). Fica num dicionário `TIPOS` no topo do arquivo, separado da lógica.
- `community=` recebe o `community_id` que o **Louvain** gravou — é o que permite
  colorir e agrupar por comunidade na tela.
- `set_edge_weight("score")` faz a **espessura da linha** representar a nota de
  similaridade que o `nodeSimilarity` calculou. Quanto mais parecidos, mais
  grossa a aresta.
- `inspector_ranking` monta a lista "casos mais similares" ao clicar num nó — é a
  funcionalidade principal do projeto aparecendo na interface.

**Sobre a organização:** toda a visualização está nesse único arquivo
(`grafo_visual.py`). Ele não fala com banco nem com interface — recebe os dados
prontos e devolve o HTML. O `graph_pipeline.py` cuida do Neo4j; o
`streamlit_app.py` cuida da tela.

**Saída:** um `.html` único, que abre sem servidor e é embutido numa aba do
protótipo.

---

## Slide 18 — Como rodar

```bash
docker compose -f docker-compose.pipeline.yml up -d   # bancos
python crud_pipeline.py                                # coleções, CRUD, agregações, Redis
python graph_pipeline.py                               # grafo + GDS
streamlit run streamlit_app.py                         # protótipo
```

A ordem importa: o grafo é construído a partir do mesmo tema de dados do
`crud_pipeline`, e o protótipo lê o que os dois produziram.

O `executar_pipeline.bat` faz os quatro passos, sobe o Docker se preciso e espera
o Neo4j aceitar conexão antes de seguir.

### Onde fica cada coisa

| Caminho | Função |
|---|---|
| `crud_pipeline.py` | MongoDB e Redis — coleções, CRUD, agregações, estruturas |
| `graph_pipeline.py` | Neo4j — monta o grafo e roda o GDS |
| `grafo_visual.py` | visualização do grafo (única parte que usa a biblioteca) |
| `streamlit_app.py` | interface |
| `logs/` | saída de cada etapa, em JSON |
| `screenshots/` | telas e coleções |
| `docs/` | modelagem (ER e JSON) |
