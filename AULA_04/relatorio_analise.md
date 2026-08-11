# Relatório Analítico: Avaliação de Estratégias de Chunking com LangChain

**1. Qual estratégia gerou mais chunks?**
A Estratégia 1 (Fixo, 200 caracteres, sem overlap). Por ter um limite de corte extremamente baixo e sem sobreposição, ela estilhaçou os documentos no maior número de fragmentos possível.

**2. Qual gerou menos chunks?**
Geralmente, a Estratégia 4 (Fixo, 2000 caracteres, sem overlap) ou a Estratégia 10 (Markdown Semântico). A estratégia de 2000 caracteres agrupa grandes volumes de texto, enquanto a Markdown cria blocos baseados apenas na ocorrência de subtítulos (que podem ser esparsos em artigos extensos).

**3. Como o tamanho dos chunks variou?**
Nas estratégias 1 a 6 (Fixo), o tamanho se manteve rígido e artificial. Nas estratégias 7 (Parágrafo) e 8 (Sentença), o tamanho variou organicamente conforme a pontuação natural do autor. Na estratégia 10 (Markdown), a variação foi extrema, dependendo do tamanho de cada seção do artigo.

**4. Qual estratégia preservou melhor a estrutura dos documentos?**
A Estratégia 10 (Markdown). Ao fatiar o texto utilizando os *headings* (H1, H2, H3) como delimitadores, ela garantiu que introduções, metodologias e conclusões não fossem misturadas, preservando metadados fundamentais.

**5. Como tabelas foram tratadas?**
Graças à utilização da biblioteca baseada em LLM na conversão, as tabelas foram preservadas em sua estrutura visual de colunas utilizando a formatação nativa do Markdown (ex: `| Coluna A | Coluna B |`). Isso manteve a relação semântica entre os dados tabulares intacta.

**6. Como imagens foram tratadas?**
O processo de OCR identificou a presença das imagens. Na conversão para texto puro/Markdown, a representação visual da imagem é descartada, mas legendas e textos contidos dentro das imagens foram extraídos e convertidos para representação textual.

**7. Quais informações foram perdidas durante a conversão PDF -> Markdown?**
Perdeu-se a formatação de layout de página complexa (como múltiplas colunas visuais), cabeçalhos e rodapés repetitivos que se misturam ao texto principal, e o significado puramente gráfico de diagramas complexos que não puderam ser convertidos em tabelas.

**8. O chunking por caracteres fragmentou conceitos ou estruturas importantes?**
Sim. Nas estratégias sem *overlap* (1 a 4), cortes artificiais muitas vezes ocorreram no meio de uma palavra, de uma tabela ou de um parágrafo crucial, destruindo o contexto exato necessário para o modelo de linguagem compreender a ideia.

**9. O chunking por parágrafo produziu chunks muito grandes?**
Sim. Como documentado nos alertas de execução (*warnings* de tamanho), textos acadêmicos frequentemente contêm blocos densos ou tabelas contínuas que não possuem quebra dupla de linha, forçando o *splitter* a gerar *chunks* maiores que o limite ideal estipulado para não quebrar a estrutura.

**10. O chunking por sentença conseguiu preservar melhor o contexto?**
Preservou o micro-contexto (as frases faziam sentido isoladamente), mas pecou no macro-contexto. Agrupar 3 sentenças de forma rígida pode quebrar um argumento que leva 5 sentenças para ser concluído pelo autor.

**11. O Recursive Splitter apresentou vantagens?**
Sim. Ao utilizar uma lista hierárquica de separadores (tentando primeiro parágrafos, depois sentenças, e só por último caracteres arbitrários), ele funcionou como a estratégia mais inteligente e balanceada para textos não-estruturados, minimizando cortes agressivos.

**12. O Markdown Splitter conseguiu preservar a estrutura semântica?**
De forma excelente. Ele não apenas manteve os blocos de seções unidos, como também embutiu metadados valiosos (informando a qual subtítulo aquele pedaço de texto pertence), o que enriquece enormemente os filtros em um banco de dados vetorial.

**13. Qual estratégia parece mais adequada para um sistema de RAG?**
A Estratégia 9 (Recursive Chunking com overlap) e a Estratégia 10 (Markdown Semântico). A estratégia recursiva é o padrão-ouro para consistência de tamanho dos vetores, enquanto a Markdown é imbatível para garantir respostas coerentes sobre tópicos específicos de um artigo.

**14. Quais estratégias devem ser descartadas?**
As Estratégias 1, 2, 3 e 4 (Fixo sem overlap). A ausência de sobreposição gera perda de contexto nas extremidades de cada *chunk*, prejudicando gravemente a capacidade da IA de recuperar informações na fronteira dos cortes.

**15. Quais estratégias você acha que devem ser utilizadas nos próximos experimentos?**
Uma abordagem híbrida: iniciar com o **Markdown Splitter** (Estratégia 10) para isolar as seções semânticas de forma macro e, em seguida, aplicar o **Recursive Character Text Splitter** (Estratégia 9) dentro de seções que ficaram muito grandes, garantindo *chunks* ricos em contexto e amigáveis ao limite de *tokens* do LLM.
