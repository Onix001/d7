# Decisões técnicas – EngMan Copilot

> Sprint 1 é **só planejamento e documentação**: nada foi implementado ainda. Todos os dados citados são **SIMULADOS**.

## 1. Critérios usados nas comparações

Usamos os mesmos 7 critérios para modelos e vector stores:

| # | Critério | O que olhamos |
|---|---|---|
| 1 | Custo | Quanto a equipe paga para usar no projeto |
| 2 | Licença/termos | Licença do software ou do modelo e o que os termos de uso permitem fazer com o nosso dado |
| 3 | Qualidade em pt-BR | Modelo: como entende e escreve em português. Vector store: se a busca funciona bem com texto em português |
| 4 | Janela de contexto | Modelo: quantos tokens cabem numa chamada. Vector store: tamanho de trecho que dá para guardar e buscar |
| 5 | Privacidade (roda local?) | Se o dado fica na nossa máquina ou vai para o servidor de outra empresa |
| 6 | Integração com Python | Biblioteca oficial e integração com LangChain |
| 7 | Maturidade/documentação | Há quanto tempo existe, quem usa e se a documentação ajuda uma equipe de 1º ano |

## 2. Modelo de linguagem (LLM)

| Critério | **Qwen 2.5 7B no Ollama local** (escolhido) | Gemini, plano gratuito (Google AI Studio) | Groq, plano gratuito |
|---|---|---|---|
| Custo | Zero, sem limite de uso (o limite é a memória do computador) | Zero dentro do limite gratuito (requisições por minuto e por dia) | Zero dentro do limite gratuito (requisições e tokens por minuto) |
| Licença/termos | Modelo aberto, licença Apache 2.0 | Termos do Google; no plano gratuito o dado enviado pode ser usado para melhorar os produtos | Termos da Groq; roda modelos abertos (ex.: Llama), cada um com a sua licença |
| Qualidade em pt-BR | Boa para um modelo pequeno; suporta tools | Muito boa; suporta tools | Boa (modelos abertos maiores); suporta tools |
| Janela de contexto | O modelo aceita bem mais que 16 mil tokens, mas no Ollama precisa configurar `num_ctx` (usamos 16.384 por causa da memória) | Muito grande (centenas de milhares de tokens) | Depende do modelo escolhido; os principais aceitam dezenas de milhares de tokens |
| Privacidade (roda local?) | **Sim.** O dado não sai da máquina | Não. O dado vai para o Google | Não. O dado vai para os servidores da Groq |
| Integração com Python | Biblioteca `ollama` e `ChatOllama` no LangChain | Biblioteca do Google e integração no LangChain | API compatível com a da OpenAI e integração no LangChain |
| Maturidade/documentação | Ollama e Qwen são bem usados e têm boa documentação; mais trabalho para instalar (~4,7 GB) | Documentação oficial completa; só precisa de uma chave de API | Documentação boa; serviço mais novo, os limites gratuitos mudam com o tempo |

- **Escolha: Qwen 2.5 7B no Ollama local.**
  - Custo zero e sem limite de uso, o que importa para rodar o eval set várias vezes.
  - Licença aberta e o dado operacional da concessionária não sai da máquina (ver [riscos e LGPD](riscos_lgpd.md), seção 1.6).
  - Suporta tools, que são a base do nosso desenho: as métricas e o cadastro vêm das tools, não da memória do modelo.
- **Trade-off aceito:**
  - O Qwen 7B é mais fraco que o Gemini em português e para raciocinar em respostas longas, tem janela menor na prática e é lento num notebook só com CPU.
  - Para compensar, as contas ficam nas tools, a temperature é baixa e as regras são curtas e testáveis.
  - Se a qualidade não for suficiente no eval da Sprint 2, a alternativa é o Gemini, só com dado SIMULADO.
- **Por que não o Ollama Cloud.** No plano da nossa conta, o uso é cobrado por token, e a proposta da equipe é um caminho 100% gratuito.

## 3. Vector store (base do RAG)

| Critério | **ChromaDB** (escolhido) | FAISS |
|---|---|---|
| Custo | Zero (código aberto) | Zero (código aberto, da Meta) |
| Licença/termos | Apache 2.0 | MIT |
| Qualidade em pt-BR | Depende do modelo de embeddings, não do banco; igual nos dois | Depende do modelo de embeddings; igual nos dois |
| Janela de contexto | Guarda o texto do trecho junto com o vetor; o tamanho do trecho (~500 tokens) quem define é o embedding | Guarda só os vetores; o texto fica em outro lugar |
| Privacidade (roda local?) | **Sim**, salvo numa pasta | **Sim**, na memória ou num arquivo |
| Integração com Python | `pip install chromadb`, biblioteca própria e LangChain | `pip install faiss-cpu` e LangChain |
| Maturidade/documentação | Documentação simples, voltada para RAG; **filtro por metadados nativo** (TAG, tipo de ativo) | Muito maduro e rápido, feito para milhões de vetores; sem filtro por metadados nativo e salvar o índice é manual |

- **Escolha: ChromaDB.** O nosso desenho depende de filtrar os trechos por `tag` e `tipo_ativo` antes da busca (bloco 3 do [fluxograma](arquitetura.png)). No ChromaDB isso é um parâmetro da busca e já vem pronto.
- **Trade-off aceito:** o FAISS é mais rápido e escala melhor, mas a nossa base tem algumas dezenas de trechos (ver o [inventário](../data/knowledge_base/inventario.md)), então a velocidade a mais não faz diferença e o filtro manual daria mais trabalho e mais chance de erro.

## 4. Embeddings

| Opção | Onde roda | Ponto forte | Ponto fraco |
|---|---|---|---|
| **nomic-embed-text** (escolhido) | Ollama local | Leve (~274 MB), aceita textos longos (até 8.192 tokens), roda no mesmo Ollama do modelo | Treinado mais em inglês |
| bge-m3 | Ollama local | Multilíngue, deve ir melhor em português | Mais pesado (~1,2 GB) |
| mxbai-embed-large | Ollama local | Boa qualidade em inglês | Aceita só ~512 tokens: um trecho de 500 fica no limite |

- **Escolha:** nomic-embed-text. É gratuito, local (mesmo motivo de privacidade do modelo) e cabe com folga o trecho de ~500 tokens. Ele pede um prefixo no texto: `search_document:` ao indexar e `search_query:` na pergunta.
- **Trade-off:** pode errar mais em perguntas escritas do jeito do técnico ("a câmera fica branca de noite"). Na Sprint 2, se a busca trouxer trechos errados nos casos do eval, a troca é pelo bge-m3. Nesse caso é preciso **indexar tudo de novo**, porque os vetores de um modelo não servem para o outro.

## 5. Orquestração

| Critério | **SDK direto do Ollama** (escolhido) | LangChain (LCEL) |
|---|---|---|
| Como funciona | Nós mesmos escrevemos o laço: chamar o modelo, ver se pediu tool, executar, devolver o resultado (até 3 rodadas) | O fluxo é montado com peças prontas ligadas por `\|` (prompt, modelo, retriever, parser) |
| Facilidade para a equipe | Pouco código e fácil de entender e depurar: dá para ver cada mensagem | Mais conceitos para aprender e mais camadas entre o código e o modelo |
| Trocar de modelo | Precisa adaptar o código para outra API | Fácil: troca `ChatOllama` por `ChatGoogleGenerativeAI` ou `ChatGroq` |
| Risco | Mais código manual | A biblioteca muda rápido entre versões e exemplos antigos deixam de funcionar |

- **Escolha:** biblioteca `ollama` direto, com o `chromadb` para a busca. Para uma equipe de 1º ano, ver e controlar cada passo do laço de tools ajuda a garantir as regras do fluxograma: no máximo 3 rodadas, timeout de 10 s e OS só depois da confirmação.
- **Trade-off:** se o projeto precisar trocar de modelo várias vezes (ex.: comparar Qwen e Gemini no eval), o LCEL economiza código. Por isso ele fica como alternativa para a Sprint 2.

## 6. Parâmetros

| Parâmetro | Valor | Por quê |
|---|---|---|
| `temperature` | 0.2 | Respostas mais consistentes e menos "criativas": o mesmo caso do eval tende a dar a mesma resposta, e o modelo inventa menos |
| `max_tokens` (`num_predict` no Ollama) | 600 | A resposta do engenheiro tem até 250 palavras (~500 tokens com as marcações); 600 dá uma margem sem deixar a resposta crescer |
| `num_ctx` | 16384 | O orçamento previsto é ~13.300 tokens (ver o [inventário](../data/knowledge_base/inventario.md), seção 4). O padrão do Ollama é bem menor, por isso o valor tem que ser configurado. Não usamos 32.768 porque a memória do contexto dobra (~0,9 GB em 16k contra ~1,9 GB em 32k para o Qwen 2.5 7B), e o notebook da equipe tem 16 GB de RAM |
| Trechos do RAG (`k`) | 4 trechos de ~500 tokens, sobreposição de 50 | ~12% da janela. A justificativa do tamanho está no inventário, seção 2 |
| Rodadas de tool | no máximo 3 por pergunta | Evita laço infinito e contexto estourado (bloco 5 do fluxograma) |
| Timeout de tool | 10 s | Regra geral do [`tools_spec.json`](../tools/tools_spec.json) |
| Histórico | últimas 6 mensagens + resumo + TAG em análise | Política de contexto do inventário, seção 4 |

**Ambiente previsto (Sprint 2):** Python 3.10 ou mais novo, no Google Colab ou no VS Code. A interface é opcional: terminal ou Gradio.
