# Inventário da base de conhecimento – EngMan Copilot

> **Todos os dados deste projeto são SIMULADOS.** Nenhum documento, TAG, número de série ou registro é da Motiva. Os fabricantes e modelos (OptiVia, Horizonte, Lumen, VozVia) são inventados.

Recorte A – Diagnóstico e Confiabilidade. Ativos cobertos: **CFTV PTZ, PMV e Callbox**.

## 1. Documentos da base

Status: **produzido** = o arquivo já está neste repositório; **a produzir** = documento planejado, que a equipe vai criar (também simulado) até a Sprint 2.

Uso: **tool** = o dado chega ao modelo só pelo retorno de uma tool ([`tools_spec.json`](../../tools/tools_spec.json)); **RAG** = o documento é dividido em trechos e indexado no ChromaDB.

| # | Documento | Descrição | Campos principais | Origem | Qtd. mínima | Status | Uso |
|---|---|---|---|---|---|---|---|
| 1 | Catálogo de ativos ([`catalogo_ativos_SIMULADO.csv`](catalogo_ativos_SIMULADO.csv)) | Cadastro de cada ativo do recorte | tag, tipo_ativo, fabricante, modelo, km, praca_portico, data_instalacao, vida_util_anos, criticidade, origem | SIMULADO, gerado pela equipe | 30 ativos (12 CFTV, 8 PMV, 10 Callbox) | produzido (30) | tool: `consultar_ativo`, `listar_ativos_por_criticidade` |
| 2 | Histórico de falhas do FSM ([`historico_falhas_SIMULADO.csv`](historico_falhas_SIMULADO.csv)) | Registros de falha e reparo, de 2025-01 a 2026-10 | id_registro, tag, data, tipo_falha, sintoma, causa_identificada, tempo_reparo_h, peca_trocada, reincidencia, condicao_ambiental, origem | SIMULADO, gerado pela equipe | 50 registros | produzido (50) | tool: `consultar_historico_falhas`, `calcular_metricas_confiabilidade` |
| 3 | Plano preventivo atual | Calendário fixo de preventivas por tipo de ativo (a situação de hoje, que o projeto quer melhorar) | tipo_ativo, atividade, periodicidade, duração, equipe | SIMULADO | 1 plano com os 3 tipos de ativo | a produzir | RAG |
| 4 | POPs (procedimentos operacionais) | Passo a passo oficial de cada intervenção, com EPI | código (POP-CFTV-01, POP-PMV-01, POP-CBX-01), tipo_ativo, tipo_intervencao, passos, EPI, riscos | SIMULADO | 3 POPs (1 por tipo de ativo) | a produzir | tool: `consultar_procedimento` |
| 5 | Fichas técnicas | Dados do fabricante de cada modelo | modelo, fabricante, alimentação, temperatura de operação, grau de proteção (IP), peças de reposição, MTBF informado | SIMULADO | 6 fichas (OV-PTZ300, HZ-PTZ200, LS-PMV3C, LS-PMV2M, VV-SOS200, VV-SOS100) | a produzir | RAG |
| 6 | Matriz de criticidade | Critérios que definem se um ativo é A, B ou C | critério, peso, exemplos, tipo_ativo | SIMULADO | 1 matriz | a produzir | RAG |
| 7 | Glossário | Siglas do setor: CFTV, PMV, SAT, SCA, CCO, CBM, RCM, WCM, FSM, RGA, LDA, λ, MTBF, MTTR, WIM, OCR | termo, definição, exemplo | Elaborado pela equipe | 16 termos | a produzir | RAG |
| 8 | Normas de segurança | Resumo, com link para o texto oficial, da NR-10 (eletricidade), da NR-35 (trabalho em altura) e da sinalização de trabalho em via | norma, item, resumo, link oficial | Resumo da equipe + link do texto oficial | 3 resumos | a produzir | RAG |
| 9 | Briefing do desafio Motiva | Resumo do problema: +5 mil km, 13 concessionárias, ~2 milhões de veículos/dia, detecção reativa e plano por calendário | tema, resumo, fonte | Material do Challenge FIAP x Motiva | 1 resumo | a produzir | RAG |
| 10 | Boletins e laudos técnicos | Relatórios de análise de falha e boletins de fabricante | TAG, data, tipo_documento, achados, recomendação, autor (cargo, sem nome) | SIMULADO | 10 documentos | a produzir | RAG |

**Uma fonte por dado:** o catálogo, o histórico e os POPs (itens 1, 2 e 4) **não** entram no RAG. Se entrassem, o modelo poderia receber duas versões do mesmo número, uma da tool e outra de um trecho antigo indexado. A regra R10 do [system prompt](../../prompts/system_prompt.md) diz o que fazer quando as fontes discordam: vale a tool, depois o documento mais novo.

## 2. Como os documentos do RAG são divididos

- **Tamanho do trecho (chunk): ~500 tokens, com 50 de sobreposição (10%).**
- **Por que 500.** A janela de contexto (Aula 02) é de 16.384 tokens (`num_ctx`) e precisa caber tudo: o prompt, as tools, o histórico e a resposta. Com 4 trechos de 500, o RAG ocupa 2.000 tokens, cerca de 12% da janela. Um trecho de 500 tokens tem mais ou menos o tamanho de uma seção de ficha técnica ou de um laudo curto, então o sintoma e a causa ficam juntos no mesmo trecho.
  - Trechos menores (~200) cortariam o laudo no meio: o sintoma ficaria num trecho e a causa em outro.
  - Trechos maiores (~1.000) deixariam caber só 2 na mesma faixa de tokens, e cada um traria mais texto que não tem a ver com a pergunta.
- **Metadados de cada trecho:** `tag` (quando o documento é de um ativo), `tipo_ativo` (CFTV, PMV ou CALLBOX), `data` (AAAA-MM-DD), `tipo_documento` (ficha, laudo, boletim, plano_preventivo, norma, glossario, briefing, matriz), além de `fonte` (nome do arquivo) e `origem` = SIMULADO.
- **Uso dos metadados:**
  - Filtro: a busca filtra por `tag` e `tipo_ativo` antes de comparar os textos, então um laudo de PMV não aparece numa pergunta sobre câmera.
  - Data: a `data` serve para a regra R10 (vale o documento mais novo).
  - Fonte: a `fonte` é o que o modelo cita na resposta (regra R4).

## 3. O que NÃO entra na base

- **Dados pessoais**, como placas, imagens e gravações das câmeras, áudio das ligações do Callbox, nome, CPF ou matrícula de empregados e dados de usuários da rodovia (LGPD, ver [`riscos_lgpd.md`](../../docs/riscos_lgpd.md)). Nos laudos, o autor aparece só pelo cargo.
- **Dados reais da Motiva**: no Sprint 1 tudo é SIMULADO.
- **Ativos fora do recorte**: SAT, SCA, WIM, OCR e iluminação.
- **Catálogo, histórico e POPs**, que chegam só pelas tools (seção 1).
- **Senhas, chaves de API e endereços internos** de rede ou de equipamentos.
- **Texto da internet sem fonte conhecida** (fórum, vídeo, "dica" sem autor).

## 4. Política de contexto

Orçamento previsto por pergunta, no pior caso (3 rodadas de tool). É o mesmo total que aparece no bloco 3 do [fluxograma](../../docs/arquitetura.png):

| Parte do contexto | Tokens (aprox.) | Como chegamos no número |
|---|---:|---|
| System prompt (sem os comentários da equipe) | 3.900 | 13.626 caracteres ÷ 3,5 |
| Definição das 6 tools | 1.950 | 6.765 caracteres ÷ 3,5 |
| Trechos do RAG | 2.000 | 4 trechos × 500 |
| Histórico da conversa | 1.900 | 3 perguntas × ~100 + 3 respostas × ~500 + resumo e TAG em análise ~100 |
| Pergunta atual, alertas do bloco 2 e pedidos de tool do modelo | 250 | pergunta ~100 + alertas ~30 + 3 pedidos de tool × ~40 |
| Retornos das tools | 2.700 | histórico com 20 registros (limite da tool) ~2.300 + métricas ~180 + POP ~125 + cadastro ~70 |
| Reserva para a resposta (`max_tokens`) | 600 | parâmetro do modelo |
| **Total** | **~13.300 de 16.384** | sobra ~3.000 (~19%) de folga |

- **Estimativa.** Os números foram calculados contando caracteres (÷ 3,5, que é uma média para texto em português). Na Sprint 2 a equipe vai conferir com a contagem real que o Ollama devolve (`prompt_eval_count`).
- **Histórico: últimas 6 mensagens + resumo.**
  - **Por que 6.** São 3 trocas de pergunta e resposta, que é o tamanho normal de um diagnóstico: pergunta, clarificação do TAG ou do sintoma, resposta. As mensagens mais antigas saem do contexto e ficam só no resumo de até 3 linhas (TAG, o que já é [FATO] e o que está pendente), que o bloco BORDA do prompt manda fazer a cada 6 mensagens. O "TAG em análise" no começo de cada resposta garante que o ativo não se perde quando as mensagens antigas saem.
  - **Alternativa 1, guardar a conversa inteira.** Cada troca usa ~600 tokens, então depois de umas 5 trocas a mais o contexto passaria da folga e o Ollama começaria a descartar as mensagens mais antigas sem avisar. Assim o modelo perderia o TAG e os fatos combinados no começo da conversa.
  - **Alternativa 2, janela sem resumo.** Mais simples, mas o modelo esqueceria o TAG e os fatos já confirmados, que é o que o eval MEM-01 testa.
- **Se passar do limite:**
  1. Primeiro o RAG cai de 4 para 2 trechos.
  2. Depois a consulta de histórico é refeita com janela menor ou filtrada por `tipo_falha`.
  3. O system prompt e a definição das tools nunca são cortados.
