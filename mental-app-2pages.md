# Assistente Interno de Conhecimento — Arquitetura RAG (Qdrant + Hybrid + HyDE)

## Objetivos do Assistente
Entregar respostas técnicas, confiáveis e citadas a partir de **milhares de PDFs**, **wikis privadas** e **tickets históricos**, em **PT-BR/ES/EN**, com **baixa latência**, **custos previsíveis** e **governança/segurança de nível corporativo**. O assistente deve: (1) **buscar** e **ranquear** trechos mais relevantes; (2) **gerar** respostas fundamentadas com **citações**; (3) respeitar **ACLs/tenant** e políticas de **privacidade**; (4) operar em **cloud** ou **on-prem**; (5) ser **observável** e **tunicável** (qualidade, custo, latência).

---

## Como capturar, fatiar e indexar esse material em um banco vetorial.

**Ingest & Normalização (guiado por necessidades do cliente)**  
Antes de tudo, mapeamos objetivos e fontes prioritárias com perguntas diretas. Ex.: órgão ambiental com milhares de PDFs de licenciamento e wikis técnicas → definimos **o que** importar, **como** está guardado e **quem** pode acessar.  
**Fluxos por tipo de documento**  
- **PDF digital (nativo):** parsers *layout-aware* (Docling, LlamaParse, Unstructured) preservando hierarquia, tabelas e ordem de leitura.  
- **PDF escaneado:** OCR (OCRmyPDF/Tesseract ou Google Doc AI/AWS Textract/Azure) → normalização por seções.  
- **Wikis (Confluence/MediaWiki):** APIs oficiais (HTML/XHTML), **delta-ingest** por `lastModified/revisions`, respeitando ACLs.  
**Higiene & Metadados**  
- Dedup/near-dup (minhash), remoção de boilerplate, **PII masking**.  
- Metadados: `tenant`, `acl_groups`, `client_id`, `doc_type`, `lang`, `created_at`, `effective_date`, `source_url`, `url#anchor`, `hash` — base p/ filtros rígidos e auditoria.

**Chunking (qualidade > quantidade)**  
- **Padrão:** janelas recursivas ~1.000 tokens (overlap 10–15%).  
- **PDFs longos:** *semantic chunking*; **Wikis:** *header-aware* por H1/H2/H3.  
- **Context-aware:** adicionar 1–2 linhas de resumo + seção-pai via LLM (pré-processado) para melhorar recall/grounding.

**Indexação (Qdrant)**  
- **Vetores nomeados:** `dense_bge_m3` (denso), `sparse_bm25/splade` (exato), opcional `colbert_tokens` (late-interaction).  
- **HNSW/ANN:** ajustar `m`, `ef_construct`; `ef` dinâmico em consulta (recall↔latência).  
- **Payload/ACL:** filtros server-side (tenant/ACL/idioma/tempo).  
- **Escala & Custo:** quantização 8-bit das embeddings; *tiering* (HNSW em RAM; payload em disco/objeto); **multi-tenancy** por coleção para isolamento e *lazy-loading*.

---

## Qual modelo de embeddings e qual tipo de LLM faria sentido (e por quê).

**Embeddings (multilíngue, híbrido)**  
- **BGE-M3** como padrão único: denso+esparso+multi-vector; ótimo PT-BR/ES/EN; habilita híbrido e caminho para **ColBERT** quando precisão cirúrgica for necessária.  
- **Quantização 8-bit** no Qdrant (economia de RAM com perda mínima).  
- **Alternativas gerenciadas:** Voyage-3-large / OpenAI text-embedding-3-large quando SaaS/SOTA justificar.

**LLM Gerador (racional técnico/custo)**  
- **Cloud enterprise:** **GPT-4o** ou **Claude Sonnet** (raciocínio, multilíngue, segurança). Decoding conservador (`temperature≤0.5`, `top_p≈0.9`, *repetition penalty* leve) e **citações**.  
- **On-prem:** **Llama-3.1** (8B/70B/405B) quantizado para soberania.  
- **Roteamento:** **Router LLM** ativa modelos de raciocínio apenas quando a complexidade exigir (pareceres/cláusulas), controlando custo/latência.

---

## Como as peças se conectam do ingest ao usuário final (descreva passo a passo).

1) **Ingest & Normalização** → parsers/OCR, limpeza, PII, metadados.  
2) **Chunking** → janelas recursivas/semantic/header-aware + contexto (1–2 linhas).  
3) **Index (Qdrant)** → `dense_bge_m3` + `sparse_bm25/splade`, HNSW, ACLs, quantização.  
4) **Chegada da Query** → **Router LLM** decide se busca é necessária e o tenant/coleção.  
5) **Parsing** → reescrita via LLM + **HyDE** (mini-documento hipotético embutido).  
6) **Pré-filtro rígido** → ACL/idioma/tempo/tipo.  
7) **Busca Híbrida** → ANN denso + BM25/SPLADE; **fusão** (RRF/pesos); **over-fetch K=30–100**.  
8) **Re-rank** → *cross-encoder* (bge-reranker/Cohere); manter **top 5–15**.  
9) **Prompt** → system prompt com regras de grounding/safety + chunks citáveis + histórico.  
10) **Geração** → LLM (cloud/on-prem) com respostas citadas e estruturadas.  
11) **Pós** → validação (JSON/esquemas), compliance/redação, localização.  
12) **Entrega** → texto + **citações clicáveis**.  
13) **Observabilidade** → traces (latência, tokens, custo), recall@K, feedback (👍/👎); coleta de *misses* p/ retuning.  
*(Opcional)* Re-iteração/Agente (nova busca, ferramentas) ou **escalonamento humano** com trilha completa.

**Diagrama mental:** Conectores → Normalização/Chunking → Embeddings → Qdrant → Router/Rewrite/HyDE → Filtro ACL → Hybrid Retrieve → Re-rank → Prompt → LLM → Pós → Usuário → Observabilidade.

---

## Como você mediria custo, latência e qualidade, além de garantir segurança e controle de acesso.

**Qualidade**  
- **Retriever:** **Recall@K** (principal), **Precision@K**, **MAP@K**, **MRR** em *golden sets* reais (10–20 prompts/domínio).  
- **Geração:** *LLM-as-judge* (fidelidade às fontes/qualidade de citação) + revisão humana em fluxos críticos.  
- **Sistema:** cobertura (≥1 chunk pertinente), thumbs up/down, escalonamentos, respostas sem fonte.

**Latência**  
- **Gargalo:** transformadores (gerador, re-ranker, rewriter).  
- **Ações:** roteamento p/ LLMs menores/quantizados em queries simples; limitar **K** (ex.: 50 → top-10); **cache semântico**; no **Qdrant**: HNSW em RAM, `ef` dinâmico, sharding por tenant, vetores **8-bit/binários** para *coarse* + *refine*.

**Custo**  
- **LLMs:** modelos menores, *caps* de tokens, prompts enxutos, endpoints dedicados/on-prem conforme escala.  
- **Vetores:** quantização, *tiering* (índice em RAM; payload em disco/objeto), multi-tenancy com carregamento seletivo; arquivar tenants inativos.  
- **Doce-ponto:** **K≈50** + **top-10 re-ranqueado** costuma otimizar custo/qualidade.

**Segurança & Acesso**  
- **AuthN/SSO + RBAC/ABAC** antes da busca e pré-prompt.  
- **Isolamento forte:** coleções por **tenant** (evitar “soft filters”).  
- **Criptografia** em trânsito/repouso; evitar enviar conteúdo sensível a LLMs externos quando política proíbe (usar on-prem).  
- **Vetores = ativos sensíveis:** mínimo acesso ao DB; hardening; ciência do risco de reconstrução textual.  
- **Política de geração:** *redaction*, bloqueios de *jailbreak*, recusar afirmações **sem fonte**.

---

## Porque essa Aplicação se encaixa no FI Group
- **Aderência ao negócio:** conteúdos **regulatórios e técnicos** (incentivos à inovação, fiscal, energia, seguros) são **multilíngues** e **PDF-intensivos** — o **híbrido denso+esparso** + **HyDE** maximiza **recall** sem abrir mão de precisão.  
- **Entrega rápida e segura:** **Qdrant** oferece **HNSW**, vetores nomeados, filtros por payload e **multi-tenancy**, facilitando **ACL forte**, isolamento por cliente e **escala elástica** (RAM/disco/objeto).  
- **Flexibilidade de implantação:** **GPT-4o/Claude** (SaaS) quando permitido; **Llama-3.1 on-prem** quando **soberania** e **compliance** exigirem. Troca de LLM é **plug-and-play** (prompt e contratos estáveis).  
- **Operação mensurável:** métricas de **qualidade/latência/custo** integradas à observabilidade (traces de ponta a ponta, *golden sets*, A/B). Time consegue iterar com segurança e provar **ROI**.  
- **Evolução orientada a risco:** começar com **chunking recursivo + BGE-M3 + híbrido + re-rank**, e acionar **ColBERT** ou **modelos de raciocínio** apenas onde o domínio exigir (jurídico, médico, auditoria), mantendo **TCO** sob controle.  
- **Valor ao cliente final:** respostas **citadas**, **consistentes** e **auditáveis**, integradas ao ecossistema FI Group, acelerando análises, reduzindo retrabalho e elevando qualidade de entregáveis.

> **Resumo**: uma única arquitetura **RAG** moderna (Qdrant + **BGE-M3** + híbrido **ANN/BM25** + **HyDE** + re-rank) com **governança**, **medidas claras** e **opções de implantação** que atendem rapidamente os diferentes cenários do FI Group — do protótipo à produção.
