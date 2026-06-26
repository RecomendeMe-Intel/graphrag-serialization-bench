# GraphRAG Serialization Benchmark

Benchmark experimental para responder uma pergunta prática sobre pipelines GraphRAG: **qual formato de serialização de subgrafo produz o melhor raciocínio em LLMs?**

Executado em GPU descentralizada (rede **Nosana**, sobre Solana), em uma NVIDIA RTX 3090, como prova de conceito para a vertical **RecomendeMe Intelligence** (OSINT / análise de redes via grafos de conhecimento).

---

## 🎯 Objetivo

Pipelines GraphRAG injetam estrutura relacional (nós e arestas de um grafo, ex. Neo4j) como contexto de prompt para um LLM raciocinar sobre conexões — em vez do RAG tradicional, que injeta apenas texto solto.

A forma como esse subgrafo é **serializado** (transformado em texto) afeta diretamente:
- a qualidade do raciocínio do modelo,
- a fidelidade às relações reais do grafo (menos alucinação),
- o custo computacional (tokens de entrada, latência).

Este experimento testa **3 formatos de serialização × 3 modelos open-source**, com avaliação automática de qualidade.

---

## 🧱 Metodologia

### Grafo de teste
Subgrafo fictício, fiel ao schema usado no RecomendeMe Intelligence (Neo4j): nós do tipo `Pessoa`, `Organização`, `Evento`, `Documento`, `Fonte`, conectados por arestas tipadas (`OPERA_EM`, `CONTATO_SUSPEITO`, `DENUNCIA` etc).

### Formatos de serialização testados

| Formato | Descrição | Risco conhecido |
|---|---|---|
| **A — Triplas** | `P001 --[OPERA_EM]--> O001 cargo=captador` (estilo RDF) | Pode não narrativizar bem |
| **B — Markdown** | Seções por tipo de nó + relações em prosa estruturada | Mais tokens; pode perder rastreabilidade da relação |
| **C — JSON** | Fiel ao schema do Neo4j | Modelo pode descrever a estrutura em vez de raciocinar sobre ela |

### Modelos testados

| Modelo | Origem | Parâmetros | Particularidade |
|---|---|---|---|
| Qwen2.5-3B-Instruct | Alibaba | 3B | Baseline |
| Mistral-7B-Instruct-v0.3 | Mistral AI | 7B | Sem gate, sem token HF |
| DeepSeek-R1-Distill-Qwen-7B | DeepSeek | 7B | Raciocínio explícito (`<think>...</think>`) |

### Pergunta enviada ao modelo (igual para todos os formatos/modelos)
1. Qual o perfil e papel do suspeito principal na rede investigada?
2. Qual evidência conecta o suspeito à vítima potencial e à organização?
3. Existe evidência de fluxo financeiro? Quem foi prejudicado?
4. Qual o nível de confiabilidade da fonte e o que ela denuncia?
5. Que lacunas de informação seriam prioritárias para investigação?

### Métricas automáticas
Score = média de 4 critérios (0 a 1), contagem de ocorrência de termos-chave na resposta:

- **citação de IDs de nó** (`P001`, `O001`...)
- **citação de tipos de relação** (`OPERA_EM`, `DENUNCIA`...)
- **distinção fato/inferência** (uso de `"Fato:"` / `"Inferência:"`)
- **menção a lacunas investigativas** (`"lacuna"`, `"falta"`, `"prioridade"`...)

> ⚠️ Alucinação **não** é medida automaticamente — exige leitura manual das respostas. O score automático mede *compliance* com o formato esperado, não necessariamente qualidade de raciocínio (ver seção de limitações).

### Infraestrutura
- **GPU**: NVIDIA RTX 3090 (24GB VRAM), alugada na rede descentralizada **Nosana** (Solana)
- **Stack**: PyTorch + HuggingFace Transformers, Jupyter
- **Dtype**: float16 para todos os modelos (Qwen, Mistral, DeepSeek)
- Sem cloud centralizada, sem API de terceiro processando os dados do grafo

---

## 📊 Resultados

| Modelo | Formato | Score | Tokens entrada | Latência |
|---|---|---|---|---|
| **Qwen 3B** | Triplas | **0.68** | 1.045 | 15.8s |
| Qwen 3B | JSON | 0.62 | 1.325 | 15.6s |
| Qwen 3B | Markdown | 0.58 | 969 | 13.5s |
| DeepSeek R1 7B | JSON | 0.59 | 1.317 | 28.7s |
| DeepSeek R1 7B | Triplas | 0.42 | 1.037 | 32.6s |
| Mistral 7B | JSON | 0.34 | 1.613 | 14.3s |
| Mistral 7B | Triplas | 0.31 | 1.236 | 18.2s |
| Mistral 7B | Markdown | 0.25 | 1.167 | 12.6s |

### Principais conclusões

1. **Qwen 3B superou os dois modelos de 7B** em todos os formatos, com menos parâmetros, menos VRAM e menor latência. Hipótese: corpus multilíngue + treinamento mais orientado a seguir instruções estruturadas (formato `Fato:`/`Inferência:`).

2. **Triplas é o formato mais eficiente globalmente** — melhor score com custo de tokens moderado no Qwen, e competitivo no Mistral. JSON consistentemente custa 25-30% mais tokens de entrada.

3. **DeepSeek preferiu JSON sobre Triplas** — possível indício de que modelos com raciocínio explícito (chain-of-thought) navegam melhor estruturas hierárquicas, enquanto modelos de instrução direta (Qwen) navegam melhor estruturas relacionais planas (triplas).

4. **Score automático ≠ qualidade real do raciocínio.** Mistral e DeepSeek zeraram no critério `distinção fato/inferência` nos formatos A e B — mas ao ler as respostas manualmente, ambos resolveram corretamente os 5 pontos da query (identificaram suspeito, vítima, fluxo financeiro, fonte e lacunas). Eles simplesmente não verbalizaram no template `"Fato:"/"Inferência:"` pedido. Isso é uma falha de *compliance de formato*, não de raciocínio sobre o grafo.

---

## ⚠️ Limitações (honestas)

- Subgrafo **fictício**, pequeno (7 nós, 10 arestas) — não reflete densidade de grafos reais de investigação
- Avaliação automática tem **viés de vocabulário**: pune o modelo por não usar o termo exato, mesmo quando o raciocínio está correto
- **Sem avaliação manual de alucinação** sistemática (apenas inspeção pontual)
- Modelos rodados em condições ligeiramente diferentes (alguns testes com tentativa de 4-bit antes de cair para float16)
- Comparação entre 3B e 7B não é "justa" em termos de escala — é deliberadamente uma pergunta sobre eficiência, não sobre capacidade máxima

## 🔭 Próximos passos

- Rodar com subgrafo real (maior densidade, força de vínculo como propriedade de aresta — `Confirmado` / `Suspeito` / `Possível`)
- Implementar métricas mais rigorosas: *Faithfulness*, *Context Recall*, *Citation Precision*, *Distinção epistêmica* (adaptado de RAGAS)
- Avaliação manual de alucinação por amostra
- Testar quantização 4-bit de forma estável para reduzir custo de VRAM em modelos 7B+

---

## 💻 Código

### Estrutura do script

```python
import os
import json
import time
import re
from dataclasses import dataclass, field

os.environ.setdefault("HF_HUB_DISABLE_XET", "1")

import torch
from transformers import AutoTokenizer, AutoModelForCausalLM

# --- Grafo de teste (fiel ao schema Neo4j) ---
SUBGRAPH = {
    "nodes": [
        {"id": "P001", "label": "Pessoa", "properties": {"nome": "Carlos Menezes", "apelido": "CarlosMZ", "status": "suspeito_ativo"}},
        {"id": "P002", "label": "Pessoa", "properties": {"nome": "Fernanda Lima", "status": "vitima_potencial"}},
        {"id": "O001", "label": "Organizacao", "properties": {"nome": "Agencia Estrela do Norte", "ativa": True}},
        {"id": "E001", "label": "Evento", "properties": {"tipo": "recrutamento_online", "data": "2024-11-03"}},
        {"id": "E002", "label": "Evento", "properties": {"tipo": "transferencia_financeira", "valor_brl": 350.00}},
        {"id": "D001", "label": "Documento", "properties": {"tipo": "print_conversa", "relevancia": "alta"}},
        {"id": "F001", "label": "Fonte", "properties": {"tipo": "denuncia_anonima", "confiabilidade": 0.82}},
    ],
    "edges": [
        {"from": "P001", "to": "O001", "type": "OPERA_EM", "properties": {"cargo": "captador"}},
        {"from": "P001", "to": "P002", "type": "CONTATO_SUSPEITO", "properties": {}},
        {"from": "P001", "to": "E001", "type": "EXECUTOU", "properties": {}},
        {"from": "E001", "to": "P002", "type": "ENVOLVEU_VITIMA", "properties": {}},
        {"from": "E002", "to": "P002", "type": "EXECUTADO_POR_VITIMA", "properties": {}},
        {"from": "E002", "to": "O001", "type": "BENEFICIOU", "properties": {}},
        {"from": "D001", "to": "E001", "type": "EVIDENCIA_DE", "properties": {}},
        {"from": "F001", "to": "P001", "type": "DENUNCIA", "properties": {}},
    ],
}

# --- 3 serializadores ---
def serialize_triples(graph):
    lines = ["=== SUBGRAFO: TRIPLAS ==="]
    for node in graph["nodes"]:
        for k, v in node["properties"].items():
            lines.append(f"{node['id']} ({node['label']}) | {k} | {json.dumps(v, ensure_ascii=False)}")
    lines.append("--- ARESTAS ---")
    for edge in graph["edges"]:
        props = " | ".join(f"{k}={v}" for k, v in edge.get("properties", {}).items())
        lines.append(f"{edge['from']} --[{edge['type']}]--> {edge['to']} {props}")
    return "\n".join(lines)

def serialize_markdown(graph):
    node_map = {n["id"]: n for n in graph["nodes"]}
    by_label = {}
    for node in graph["nodes"]:
        by_label.setdefault(node["label"], []).append(node)
    lines = ["# Subgrafo de Investigacao"]
    for label, nodes in by_label.items():
        lines.append(f"\n## {label}s")
        for node in nodes:
            lines.append(f"\n### {node['id']}")
            for k, v in node["properties"].items():
                lines.append(f"- **{k}**: {json.dumps(v, ensure_ascii=False)}")
    lines.append("\n## Relacoes")
    for edge in graph["edges"]:
        src = node_map[edge["from"]]["properties"].get("nome") or edge["from"]
        dst = node_map[edge["to"]]["properties"].get("nome") or edge["to"]
        lines.append(f"- **{src}** `{edge['type']}` **{dst}**")
    return "\n".join(lines)

def serialize_json(graph):
    return "=== SUBGRAFO JSON ===\n" + json.dumps(graph, ensure_ascii=False, indent=2)

SERIALIZERS = {"A_triplas": serialize_triples, "B_markdown": serialize_markdown, "C_json": serialize_json}

# --- Prompt fixo ---
SYSTEM_PROMPT = """Voce e um analista de inteligencia especializado em investigar redes de trafico de pessoas no Brasil.
Responda EXCLUSIVAMENTE com base no contexto fornecido.
Regras:
1. Cite IDs dos nos e tipos de relacao (ex: P001 OPERA_EM O001).
2. Use "Fato:" para dados do grafo e "Inferencia:" para deducoes.
3. Se nao houver dado, diga: "Nao ha dados no subgrafo sobre isso."
4. Responda em portugues brasileiro."""

QUERY = """Com base no subgrafo, responda:
1. Qual o perfil e papel do suspeito principal na rede investigada?
2. Qual evidencia conecta o suspeito a vitima potencial e a organizacao?
3. Existe evidencia de fluxo financeiro? Quem foi prejudicado?
4. Qual o nivel de confiabilidade da fonte e o que ela denuncia?
5. Que lacunas de informacao seriam prioritarias para investigacao?"""

# --- Avaliação automática (compliance, não qualidade absoluta) ---
EVAL_CRITERIA = {
    "citacao_ids": ["P001", "P002", "O001", "E001", "E002", "D001", "F001"],
    "citacao_relacoes": ["OPERA_EM", "CONTATO_SUSPEITO", "EXECUTOU", "EVIDENCIA_DE", "DENUNCIA", "BENEFICIOU"],
    "distincao_fato_inferencia": ["Fato:", "Inferencia:"],
    "resposta_lacunas": ["lacuna", "ausente", "nao ha dados", "falta", "prioridade"],
}

@dataclass
class BenchmarkResult:
    format_name: str
    response: str
    latency_s: float
    input_tokens: int
    output_tokens: int
    scores: dict = field(default_factory=dict)

    def score_total(self):
        vals = [v for v in self.scores.values() if v is not None]
        return sum(vals) / len(vals) if vals else 0.0

def build_prompt(context, tokenizer):
    messages = [
        {"role": "system", "content": SYSTEM_PROMPT},
        {"role": "user", "content": f"CONTEXTO:\n{context}\n\n---\n\n{QUERY}"},
    ]
    return tokenizer.apply_chat_template(messages, tokenize=False, add_generation_prompt=True)

def run_inference(prompt, tokenizer, model, max_new_tokens=1000):
    inputs = tokenizer(prompt, return_tensors="pt").to(model.device)
    input_len = inputs["input_ids"].shape[1]
    t0 = time.perf_counter()
    with torch.no_grad():
        outputs = model.generate(**inputs, max_new_tokens=max_new_tokens, do_sample=False, pad_token_id=tokenizer.eos_token_id)
    latency = time.perf_counter() - t0
    generated = outputs[0][input_len:]
    return tokenizer.decode(generated, skip_special_tokens=True), latency, input_len, len(generated)

def evaluate(result):
    resp = result.response.lower()
    for criterion, tokens in EVAL_CRITERIA.items():
        hits = sum(1 for t in tokens if t.lower() in resp)
        result.scores[criterion] = round(hits / len(tokens), 2)
    return result

def run_benchmark(model_id, max_new_tokens=1000):
    tokenizer = AutoTokenizer.from_pretrained(model_id)
    model = AutoModelForCausalLM.from_pretrained(model_id, torch_dtype=torch.float16, device_map="auto")
    model.eval()

    results = []
    for fmt_name, serializer in SERIALIZERS.items():
        context = serializer(SUBGRAPH)
        prompt = build_prompt(context, tokenizer)
        response, latency, in_tok, out_tok = run_inference(prompt, tokenizer, model, max_new_tokens)
        result = evaluate(BenchmarkResult(fmt_name, response, latency, in_tok, out_tok))
        results.append(result)
        print(f"{fmt_name} | score={result.score_total():.2f} | latencia={latency:.1f}s | tokens_in={in_tok}")

    return results
```

### Como rodar (Jupyter / Nosana)

```python
# Célula 1 — instalar dependências
!pip install transformers accelerate -q

# Célula 2 — criar o arquivo do script (cole o código acima)
%%writefile graphrag_benchmark.py
# ... conteúdo completo do script ...

# Célula 3 — rodar
import graphrag_benchmark as bench
bench.run_benchmark("Qwen/Qwen2.5-3B-Instruct")
# ou
bench.run_benchmark("mistralai/Mistral-7B-Instruct-v0.3")
# ou (modelo com <think>...</think> — requer extração separada do raciocínio)
bench.run_benchmark("deepseek-ai/DeepSeek-R1-Distill-Qwen-7B")
```

> 💡 Para o DeepSeek-R1-Distill, adicione uma função de extração do bloco `<think>...</think>` antes de avaliar a resposta, já que o modelo expõe o raciocínio em cadeia separado da resposta final.

---

## 🧩 Por que infraestrutura descentralizada

A vertical RecomendeMe Intelligence trabalha com dados de investigação sensíveis (OSINT, denúncias, perfis de risco). Rodar inferência em GPU alugada na rede Nosana, em vez de API de terceiro centralizado (OpenAI, AWS Bedrock etc.), significa que:

- o prompt e os dados do grafo nunca passam por servidor de uma big tech;
- a inferência acontece localmente no nó que processa o job;
- não há dependência de termos de serviço de cloud centralizada para um caso de uso que já é fiscalizado externamente (denúncias, MPF).

Este benchmark é a validação de que esse pipeline é tecnicamente viável antes de qualquer decisão de arquitetura de produção (Ollama vs vLLM, escolha final de modelo, dimensionamento de infra).

---

## 📁 Arquivos de resultado

- `benchmark_results.json` — Qwen2.5-3B-Instruct
- `benchmark_results_mistral.json` — Mistral-7B-Instruct-v0.3
- `benchmark_results_deepseek.json` — DeepSeek-R1-Distill-Qwen-7B (inclui campo `thinking` com o raciocínio em cadeia)

Cada arquivo contém, por formato testado: score, scores detalhados por critério, latência, tokens de entrada/saída e a resposta completa do modelo.
