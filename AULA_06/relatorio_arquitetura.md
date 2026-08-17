# Projeto de arquitetura rag: cenários de inteligência de ameaças e análise de apólices

## Parte 1 - Identificação dos problemas

### Cenário 1: Assistente de inteligência de ameaças (CyberOps RAG)

**1.1 Descrição do problema**

* **Problema:** Analistas de segurança gastam muito tempo cruzando manualmente logs de varredura de rede com extensos bancos de dados de vulnerabilidades (CVEs) para identificar riscos.
* **Usuário:** Analistas de *Security Operations Center* (SOC) e *pentesters*. Nível técnico avançado, familiarizados com ambientes Linux (Ubuntu, Kali), redes e ferramentas de linha de comando.
* **Informação consultada:** Saídas de *scans* (Nmap, WHOIS, relatórios de *pentest*) cruzados com o banco de dados oficial de CVEs (*Common Vulnerabilities and Exposures*).
* **Origem da informação:** Arquivos de log gerados localmente pelas ferramentas de varredura e *feeds* JSON diários do banco nacional de vulnerabilidades (NVD).
* **Por que LLM puro não serve:** Modelos pré-treinados alucinam versões de *softwares*, inventam IDs de CVEs inexistentes e não possuem conhecimento da topologia interna da rede da empresa (que é confidencial).
* **Interface:** Integração via API diretamente no terminal (CLI) ou painel de SIEM.
* **Perguntas reais:**
1. *"Quais CVEs críticas de 2025 afetam o serviço Apache rodando na porta 443 relatado no log do IP 192.168.1.50?"*
2. *"Existe algum exploit conhecido para as versões de SSH listadas no scan da sub-rede de homologação?"*
3. *"Como aplicar o patch de mitigação para a vulnerabilidade detectada na máquina SRV-DATABASE ontem?"*



**1.2 Por que RAG?**

* **Adequação:** O RAG permite injetar o contexto exato do *log* do dia e a base de CVEs atualizada na memória do LLM, garantindo respostas baseadas na infraestrutura real.
* **Conhecimento:** Arquivos de logs locais e metadados técnicos de *exploits*.
* **Frequência:** Altíssima (diária/em tempo real). Novos relatórios de rede são gerados a cada hora e novas CVEs são publicadas diariamente.
* **Documentos privados:** Sim, a topologia de rede e os IPs expostos da organização são informações altamente críticas e sigilosas.
* **Risco do pré-treino (Exemplo):** O LLM poderia afirmar que a porta 8080 do servidor interno está vulnerável à vulnerabilidade "Log4Shell", instruindo o analista a derrubar o serviço, quando na verdade o log interno mostra que a aplicação rodando ali é nativa e imune, causando indisponibilidade no negócio (falso positivo).

**1.3 Limitações - quando RAG não é a resposta**

* **Quando não usar:** Quando o objetivo é análise quantitativa ou detecção de anomalias estatísticas.
* **Alternativas:**
* *Busca por palavra-chave:* Excelente se o analista já sabe o ID exato (ex: "CVE-2024-1234").
* *Banco de dados relacional (SQL/SIEM):* Fundamental para consultas como "Mostre todos os IPs com a porta 22 aberta".
* *Regras determinísticas:* Essencial para bloqueios imediatos de firewall (ex: IP em *blacklist* detectado = bloqueio automático, sem consultar IA).


* **Pergunta que o RAG responde mal:** *"Qual é a média de portas abertas por máquina na nossa rede corporativa?"*. O RAG recuperaria trechos de texto, mas não sabe calcular médias sobre bases massivas. Um banco de dados SQL com `AVG(portas)` faria isso em milissegundos.
* **Contar/Somar/Ordenar:** Se o usuário pedir para contar informações espalhadas, o RAG falhará severamente, pois ele não recupera todos os documentos (apenas os *Top K* mais similares). O resultado será uma contagem parcial e incorreta.

---

### Cenário 2: Oráculo de apólices e procedimentos (Admin RAG)

**1.1 Descrição do problema**

* **Problema:** Dificuldade e lentidão na interpretação de contratos de seguro longos e cheios de jargões técnicos para aprovação ou recusa de sinistros.
* **Usuário:** Assistentes administrativos e operadores de atendimento de seguradoras. Perfil técnico focado em regras de negócio e rotinas de escritório.
* **Informação consultada:** Condições gerais, limites de indenização, tabelas de carência e cláusulas de exclusão.
* **Origem da informação:** Manuais em PDF e contratos digitais (DOCX) fornecidos pelas áreas jurídica e de produtos da empresa.
* **Por que LLM puro não serve:** O LLM conhece as leis gerais de seguros, mas desconhece os produtos privados e os limites financeiros específicos da apólice comercializada por aquela empresa específica.
* **Interface:** Interface Web (dashboard interno) ou Chatbot no sistema de CRM.
* **Perguntas reais:**
1. *"O seguro residencial básico plano 2026 cobre queima de placa solar por raio?"*
2. *"Qual é o prazo de carência para acionar a cobertura de desemprego involuntário?"*
3. *"A cláusula de exclusão 4.1 se aplica a motoristas de aplicativo no seguro auto frota?"*



**1.2 Por que RAG?**

* **Adequação:** Transforma a busca estática em páginas densas de PDF em uma resposta direta, traduzindo o "juridiquês" com citação exata da página.
* **Conhecimento:** Regras de negócio restritas, tabelas de valores e condições legais.
* **Frequência:** Baixa (atualizações anuais ou quando há mudança regulatória).
* **Documentos privados:** Sim, produtos estratégicos da seguradora não abertos ao mercado concorrente.
* **Risco do pré-treino (Exemplo):** O LLM poderia responder: *"Sim, danos elétricos têm cobertura imediata"*, baseando-se em práticas de mercado, ignorando que o produto específico da empresa exige 30 dias de carência. Isso geraria um passivo financeiro e legal para a seguradora.

**1.3 Limitações**

* **Alternativas e combinações:** A melhor abordagem aqui é uma combinação de **RAG com filtros estruturados (SQL)**. Para regras exatas de preço ("Quanto custa a franquia?"), regras determinísticas no sistema legado superam o RAG.

---

## Parte 2 - Organização dos documentos

### Cenário 1 (CyberOps RAG)

* **Tipos de arquivo:** `.log`, `.txt`, `.json` e `.xml`.
* **Volume:** Milhares (crescimento rápido).
* **Tamanho:** Leves (dezenas a poucas centenas de KB por log).
* **Frequência:** Ingestão contínua. Documentos de *scans* antigos perdem relevância e podem ser expurgados ou arquivados (rotação de logs).
* **Organização de pastas:**
```text
documentos/
├── infra_interna/
│   ├── nmap_scans/
│   └── relatorios_pentest/
├── threat_intelligence/
│   ├── cves_json/
│   └── feeds_externos/

```


*Justificativa:* Separa claramente dados de inteligência global (CVEs) dos dados confidenciais locais (infra_interna). Facilita aplicar filtros estritos na busca ("pesquise apenas nos scans locais").
* **Sigilo e controle:** Senhas em texto plano capturadas em relatórios de *pentest* **não** devem entrar na base. A prevenção é feita por *scripts* de anonimização com expressões regulares (Regex) durante a extração, mascarando formatos que pareçam *hashes* ou senhas.
* **Controle de versão:** Logs não são versionados, são temporais. A busca deve filtrar obrigatoriamente pela data mais recente disponível.

### Cenário 2 (Admin RAG)

* **Tipos de arquivo:** `.pdf`, `.docx`.
* **Volume:** Centenas.
* **Tamanho:** Pesados estruturalmente (manuais de 50 a 200 páginas, 1MB a 15MB).
* **Frequência:** Lotes anuais. Arquivos não são apagados, pois sinistros antigos seguem a regra do ano de contratação.
* **Organização de pastas:**
```text
documentos/
├── auto/
│   ├── 2024/
│   ├── 2025/
│   └── 2026/
├── residencial/
├── vida/
└── circulares_internas/

```


*Justificativa:* O usuário pensa em categorias de produto e ano de vigência. Se o sinistro é de um carro de 2025, o sistema isola o contexto nas pastas `/auto/2025/`.
* **Sigilo e Controle:** Apólices assinadas contendo dados sensíveis de clientes (CPF, endereço) devem ser barradas (adequação à LGPD). A base de conhecimento deve conter apenas os contratos *em branco* (modelos gerais).
* **Controle de Versão:** Resolve-se com a estrutura de pastas e metadados de `ano_vigencia`. O filtro RAG (`where ano == "2024"`) impede que a apólice de 2026 contamine a resposta.

---

## Parte 3 - Pipeline de ingestão

### 3.1 Extração

* **Cenário 1 (CyberOps):** Leitura direta via Python utilizando bibliotecas JSON nativas e manipulação de *strings* para varrer logs.
* **Cenário 2 (Admin):**
* *PDFs textuais:* Uso do `pymupdf4llm` para preservar a semântica em Markdown.
* *PDFs escaneados:* Aplicação de OCR (ex: Tesseract) para extrair o texto impresso.
* *Tabelas:* Conversão para tabelas Markdown (`| Coluna |`), essencial para limites de indenização.
* *Imagens/Multimodal:* Logotipos e ilustrações são descartados. Tabelas em formato de imagem passam por OCR multimodal (Modelos de visão).


* *Problema prático:* Ao usar conversores comuns, tabelas com colunas mescladas quebram e misturam valores financeiros. O uso de *parsers* focados em Markdown evita que R$ 1.000 de franquia caia na linha da cobertura básica.

### 3.2 Limpeza e normalização

* **O que remover:** Cabeçalhos/rodapés repetidos em todas as páginas (poluem a semântica vetorial) e índices/sumários (geram falsos positivos de contexto).
* **O que padronizar:** Remoção de quebras de linha no meio de frases (`\n` acidentais), e normalização de codificação para UTF-8.
* **Risco de limpar demais:** No CyberOps, remover caracteres especiais pode destruir a sintaxe de um código de *exploit* contido na CVE.

### 3.3 Frequência de ingestão

* **CyberOps:** Agendado em *cronjobs* a cada hora. Apenas os logs novos gerados na última hora são vetorizados (processamento incremental).
* **Admin:** Sob demanda (quando um produto novo é lançado). O reprocessamento afeta apenas o PDF novo anexado, reconhecido pelo *hash* do arquivo no banco de dados.

---

## Parte 4 - Metadados

### Schemas

**Schema - CyberOps RAG (Log Chunk)**

```json
{
  "document_id": "scan_nmap_srv01",
  "chunk_id": "scan_nmap_srv01_blk3",
  "ferramenta": "nmap",
  "ip_alvo": "192.168.1.50",
  "data_scan": "2026-08-14",
  "criticidade_geral": "alta",
  "text": "PORT 443/tcp open ssl/http Apache 2.4.49..."
}

```

*Justificativa:* O `ip_alvo` e a `data_scan` são cruciais. Sem isso, vetores de máquinas diferentes se misturariam na análise.

**Schema - Admin RAG (Apolice Chunk)**

```json
{
  "document_id": "cond_gerais_auto_26",
  "chunk_id": "cg_auto_26_sec4",
  "produto": "seguro_auto",
  "ano_vigencia": 2026,
  "secao_juridica": "Cláusula 4 - Exclusões",
  "pagina": 12,
  "text": "..."
}

```

*Justificativa:* `produto` e `ano_vigencia` formam os filtros primários (hard-filters). `pagina` e `secao_juridica` são usados para a citação obrigatória da fonte.

### Filtros e questões estratégicas

* **Exemplo de filtro:** Na pergunta *"Qual a cobertura do seguro auto 2024?"*, o RAG é forçado a buscar `where metadata.produto == 'seguro_auto' AND metadata.ano_vigencia == 2024`.
* **Citação na tela (Admin):** *Fonte: Condições Gerais Seguro Auto, Edição 2026 - Cláusula 4 - Exclusões (Página 12).*
* **Metadado caríssimo pós-indexação:** Uma *tag semântica manual* (ex: "tipo_cobertura: perda_total"). Se esquecermos de criar isso no início, teríamos que passar todos os documentos por um LLM estruturador novamente, pagar pelo processamento e recriar toda a matriz de *embeddings* vetoriais do zero.

---

## Parte 5 - Chunking / Splitting

### Estratégias e Justificativas

**Cenário 1 (CyberOps)**

* **Estratégia:** Divisão baseada em caracteres/linhas lógicas. Para *logs*, um separador customizado (ex: nova linha iniciada com `IP: ` ou `PORT `) mantém o contexto de uma máquina isolado. Para CVEs, JSON Splitter focado nos nós da estrutura.
* **Tamanho e Overlap:** Tamanhos flexíveis (um *host* no nmap pode ter 5 ou 50 portas abertas). *Overlap* deve ser 0 (zero) entre máquinas diferentes para não vazar portas do IP "A" para o *chunk* do IP "B".

**Cenário 2 (Admin)**

* **Estratégia:** Splitter Recursivo Semântico (`MarkdownHeaderTextSplitter` seguido de um recursivo padrão). Quebra por Seções (Títulos H2, H3). Um contrato e uma transcrição de call center exigem abordagens totalmente distintas (contratos precisam respeitar hierarquia de cláusulas; falas de áudio quebram por turno de falante).
* **Tamanho e Overlap:** ~800 a 1000 caracteres, pois as cláusulas jurídicas costumam ter de 3 a 5 parágrafos. *Overlap* de 150 a 200 caracteres para garantir que a ressalva "Exceto nos casos de..." do parágrafo seguinte não seja separada da regra principal.
* **Problemas de tamanho:**
* *Muito pequeno:* Perde o contexto (a frase "Exceto em roubo" sozinha não significa nada se o objeto do roubo ficou no *chunk* anterior).
* *Muito grande:* Polui o prompt do LLM, aumenta custos e traz regras de outras apólices irrelevantes, confundindo a IA.


* **Tratamento de tabelas:** Uma tabela cortada ao meio perde os cabeçalhos das colunas e destrói o referencial de dados. Ela deve ser encapsulada integralmente em um único *chunk* ou, se gigantesca, sumarizada previamente por um LLM gerando um *chunk* descritivo adicional.
* **Evidência de bom chunking:** Testes cegos de *Retrieval*. Fazer perguntas complexas e medir se os top-3 documentos retornados pelo banco vetorial contêm a resposta exata de forma íntegra.

---

## Parte 6 - Embeddings

| Item | Modelo Escolhido | Dimensão | Suporta PT? | Multilíngue? | Tam. Máximo (Tokens) | Open Source? | Local? | API? | Custo Aproximado | Fonte (Link) |
| --- | --- | --- | --- | --- | --- | --- | --- | --- | --- | --- |
| **CyberOps** | `jina-embeddings-v2-base-code` | 768 | Parcialmente | Sim (Inglês predominante) | 8192 | Sim | Sim | Sim | Gratuito (Local) | [HuggingFace - Jina AI](https://huggingface.co/jinaai/jina-embeddings-v2-base-code) |
| **Admin** | `text-embedding-3-large` | 3072 | Sim (Nativo) | Sim | 8191 | Não | Não | Sim | U$0.13 / 1M tokens | [OpenAI API Docs](https://platform.openai.com/docs/guides/embeddings) |

**Justificativas em texto:**

* Para o **CyberOps**, documentos técnicos, códigos e *logs* (predominantemente em inglês ou linguagem de máquina) exigem um modelo treinado em sintaxe computacional. O `jina-embeddings-v2-base-code` é *open source*, altamente capaz com código e roda 100% local (essencial, pois dados de vulnerabilidade da rede não podem ser enviados para APIs externas por risco de vazamento).
* Para o **Admin**, o vocabulário jurídico denso em português brasileiro exige o ápice da compreensão semântica. O `text-embedding-3-large` da OpenAI lida perfeitamente com nuances do idioma. Como as apólices em branco (modelos) não possuem dados sensíveis (LGPD), a API de baixo custo atende perfeitamente.

**Respostas Analíticas:**

* *Alternativas descartadas:* O `all-MiniLM-L6-v2` foi descartado para o cenário de seguros porque, embora rápido, sua capacidade de capturar nuances em textos muito densos e em português é inferior a modelos maiores ou de última geração.
* *Tamanho de entrada e Chunking:* O limite de tokens (ex: 8192 do Jina) dita o teto arquitetural. Se nossos *chunks* da Parte 5 superassem esse limite, o modelo truncaria o final do texto cegamente, gerando vetores incompletos. Nossos *chunks* propostos (1000 caracteres) cabem com enorme folga nesse limite.

---

## Parte 7 - Arquitetura final

### Diagramas de arquitetura

**Arquitetura cenário 1 (CyberOps RAG) - Local/Segura**

```text
[Logs Nmap/WHOIS] + [CVE Feeds]
       │
       ▼ (Python Custom Scripts)
[Limpeza de Timestamps e Regex]
       │
       ▼ (Logical Splitting)
[Chunks por IP/Porta & CVE] ---> [Metadados: IP, Data, Ferramenta]
       │
       ▼ (Jina v2 Code - Local GPU)
[Vetorização Embeddings]
       │
       ▼
[Vector Store Local (Milvus/ChromaDB)]
       ^
       │ (Hybrid Search: Semântica + Palavra-chave)
       ▼
[LLM Local (Ex: Llama 3)] <--- Prompt Seguro (Temperatura 0)
       │
       ▼
[Terminal do Analista SOC]

```

**Arquitetura cenário 2 (Admin RAG) - Nuvem/API**

```text
[Manuais e Contratos PDF]
       │
       ▼ (pymupdf4llm / OCR)
[Markdown Estruturado (Tabelas Preservadas)]
       │
       ▼ (Markdown Header Text Splitter)
[Chunks Hierárquicos (Seções)] ---> [Metadados: Produto, Ano, Página]
       │
       ▼ (OpenAI text-embedding-3-large via API)
[Vetorização Embeddings]
       │
       ▼
[Vector Store Cloud (Pinecone/FAISS)]
       ^
       │ (Semantic Search + Hard Filters de Ano/Produto)
       ▼
[LLM (Ex: GPT-4o)] <--- Prompt com exigência de Citação de Fonte
       │
       ▼
[Dashboard Administrativo Web]

```

### Tabela de decisões consolidada

| Etapa | Decisão (CyberOps) | Justificativa | Decisão (Admin) | Justificativa |
| --- | --- | --- | --- | --- |
| **Extração** | Scripts Python JSON/Txt | Maior agilidade e parsing nativo. | OCR + Markdown Parser | Necessário para preservar tabelas e cabeçalhos visuais. |
| **Limpeza** | Remoção de *timestamps* isolados | Otimiza os *tokens* focando nas vulnerabilidades. | Remoção de cabeçalhos/rodapés repetitivos | Evita falsos positivos por repetição de título na busca. |
| **Chunking** | Lógico (por IP/Seção) com *overlap* 0 | Evita misturar falhas de máquinas diferentes. | Hierárquico (~1000 char, *overlap* 200) | Mantém a coesão entre cláusulas longas e exceções legais. |
| **Metadados** | Data, IP Alvo, Ferramenta | Essencial para rastrear a topologia real da rede. | Produto, Ano Vigência, Página | Essencial para citar fontes e não misturar contratos. |
| **Embeddings** | Open Source Local (Jina-Code) | Segurança estrita (dados confidenciais) e jargão de rede. | API Proprietária (OpenAI Text-3) | Maior precisão semântica em jargões jurídicos em português. |

**Limitações da Proposta:** Esta arquitetura administrativa não permite aprovações transacionais (o RAG informa a regra, mas não aperta o botão de pagar o sinistro). No cenário Cyber, o RAG é dependente da formatação dos logs; se a versão do Nmap mudar drasticamente o *output*, os extratores quebrarão.

---

## Parte 8 - Comparação entre os dois cenários

* **Diferenças:** As decisões de Ingestão e Vetorização foram diametralmente opostas. O CyberOps priorizou modelos de *código* e processamento local por sigilo de rede, usando quebra lógica (zero overlap). O Admin RAG priorizou modelos semânticos avançados via API e fatiamento baseado em texto corrido (Markdown com overlap) para lidar com PDFs longos. O contraste de metadados também é notório (temporal vs. categórico).
* **Semelhanças:** Ambos compartilham a base arquitetural do RAG (*Vector Store* -> *Retrieval* com base no *Prompt*) e ambos exigem **Filtragem de Metadados (Hard Filtering)** antes da busca semântica para evitar catástrofes (misturar IPs em um, misturar produtos em outro). Isso sinaliza que o filtro de metadados é uma boa prática geral, e não repetição mecânica.
* **Escolha pessoal:** Eu escolheria construir o **Cenário 1 (CyberOps RAG)**. Como estudante de Ciência de Dados focado em arquitetura e automação local em distribuições Linux, orquestrar a ingestão de dados de infraestrutura e rodar modelos localmente utilizando o limite computacional da GPU representa um desafio técnico mais profundo e imediato do que o parseamento comercial de apólices, além de ter impacto imediato na detecção e remediação ativa de ameaças reais.

---

## Ferramentas de IA utilizadas

Durante a estruturação mental e arquitetural desta entrega, o Google Gemini foi utilizado como suporte cognitivo. As interações foram focadas em solicitar simulações de cenários de estresse para as estratégias de *chunking* e extração em tabelas complexas de PDF, validar conceitos da infraestrutura *open source* do HuggingFace e auditar a estruturação das tabelas comparativas para atender aos critérios exigidos pela rubrica de avaliação. O conteúdo final foi revisado tecnicamente considerando fundamentos práticos de projetos RAG desenvolvidos ao longo do módulo.
