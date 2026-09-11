# Revisão de índices e desempenho

## Lista de verificação da consulta

1. Confirmar a consulta exata e o padrão de parâmetros.
2. Inspecionar `SHOW CREATE TABLE` e `SHOW INDEX` antes de propor índices.
3. Revisar tipo de acesso, chave escolhida, tamanho da chave, linhas examinadas/estimadas, percentual filtrado, loops e tempo real quando disponíveis.
4. Identificar predicados por função: igualdade, range, junção, ordenação/agrupamento e projeção.
5. Verificar sargabilidade e compatibilidade de tipos.
6. Avaliar distribuição e seletividade dos dados em vez de confiar apenas no nome das colunas.
7. Considerar frequência da consulta e frequência de escrita antes de adicionar índice.

## Índices compostos

- Predicados de igualdade normalmente são bons candidatos para posições iniciais, mas a ordem das colunas deve refletir o caminho completo de acesso.
- Um predicado de range frequentemente limita o uso das partes seguintes da chave para redução de linhas, embora partes posteriores ainda possam ajudar em filtragem/cobertura dependendo da versão e do plano.
- Considerar compatibilidade de `ORDER BY`/`GROUP BY` com filtros e direção de ordenação.
- Um predicado único ou altamente seletivo não precisa ficar automaticamente primeiro se outra ordenação atender melhor à carga de trabalho completa.
- Validar com o plano em vez de depender somente de heurísticas de livro.

## Verificação de redundância

Antes de propor `INDEX(a,b)`, inspecionar se já existem índices como:

- `INDEX(a,b)` — duplicata exata.
- `INDEX(a,b,c)` — pode já cobrir o novo índice por prefixo à esquerda.
- `UNIQUE(a,b)` — pode tornar um índice não único `(a,b)` redundante para busca.
- outro índice com colunas iniciais compatíveis, mas finalidade diferente — comparar planos reais e carga de trabalho antes de decidir.

Não remover automaticamente índices menores após criar índices maiores. Um índice menor pode ocupar menos espaço, ser mais barato para varredura ou atender carga diferente.

## Chaves primárias no InnoDB

- O InnoDB armazena as linhas clusterizadas pela chave primária.
- Índices secundários carregam as colunas da chave primária como localizador da linha.
- Chaves primárias largas aumentam o tamanho dos índices secundários.
- Chaves de inserção aleatórias podem aumentar divisões de página, fragmentação, pressão sobre conjunto de trabalho e custo de escrita.
- Preferir chaves estreitas e estáveis quando a arquitetura permitir, considerando requisitos de IDs distribuídos/negócio.

## Anti-padrões comuns a investigar

- `SELECT *` quando apenas poucas colunas são necessárias.
- `WHERE FUNCTION(indexed_column) = ...` quando uma reescrita por range pode preservar o uso do índice.
- `LIKE '%value%'` esperando que um B-tree normal acelere o curinga inicial.
- tipos/collations de string ou signedness incompatíveis em junções.
- conversão implícita entre número e string.
- paginação com `OFFSET` grande em páginas profundas; considerar paginação por chave/seek quando apropriado.
- `DISTINCT` desnecessário mascarando problema de junção.
- subconsultas correlacionadas executadas em alta cardinalidade.
- conjuntos de resultados enormes quando transferência/serialização domina o tempo do banco.

## Interpretação conservadora de planos

Não marcar como defeito apenas pelo nome:

- `ALL`/varredura de tabela: pode ser ótimo para tabelas pequenas ou leituras de baixa seletividade.
- `Using filesort`: descreve a estratégia de ordenação do MySQL, não necessariamente I/O em disco nem bug de desempenho.
- `Using temporary`: pode ser aceitável dependendo da quantidade de linhas e frequência da consulta.

Focar em custo medido, linhas processadas, loops, latência e frequência da carga.

## Validação antes/depois

Registrar, quando possível:

- valores representativos dos parâmetros;
- correção do resultado da consulta;
- tempo de execução sob condições comparáveis;
- linhas examinadas versus linhas retornadas;
- plano de execução e índice selecionado;
- efeitos de buffer/cache ao interpretar tempos;
- tamanho do índice e overhead de escrita;
- impacto na carga de trabalho, não apenas em uma consulta isolada.

Não usar um único microbenchmark com cache aquecido como prova de melhoria em produção.
