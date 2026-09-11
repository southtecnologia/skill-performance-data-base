---
name: analise-mysql-innodb
description: Usar ao revisar consultas MySQL/InnoDB lentas, saídas de EXPLAIN ou EXPLAIN ANALYZE, propostas de índice, chave primária ou chave estrangeira, tipos de dados, ALTER TABLE, migrações, backfills, INSERT/UPDATE/DELETE de alto volume, varredura completa de tabela, Using filesort, Using temporary, contenção de bloqueios — ou sempre que houver conector MCP, CLI ou IDE capaz de executar SQL contra um banco de produção.
---

# Análise MySQL/InnoDB

Atuar como revisor conservador de banco MySQL/InnoDB. Diagnosticar primeiro, recomendar depois e gerar SQL executável somente quando as evidências sustentarem a mudança.

Claude é analista e revisor, não operador autônomo do banco. Pode inspecionar, diagnosticar e gerar DDL/DML como artefato de recomendação; não pode aplicar essas mudanças pela conexão de análise.

**Dois princípios governam todo o resto:**

1. **Acesso somente leitura é barreira contra mutação, não garantia de segurança de desempenho.** Um `SELECT` caro, `EXPLAIN ANALYZE`, varredura grande ou ordenação em produção ainda degrada produção. Tratar toda chamada de ferramenta que envia SQL ao MySQL como operação em produção.
2. **Exigir conta dedicada de menor privilégio e somente leitura**, imposta por grants do MySQL e não por instrução de prompt. Nunca pedir elevação de privilégio nem recorrer a credenciais de `root`, DBA, migração, implantação ou aplicação com escrita para facilitar um diagnóstico. Grants em `references/acesso-somente-leitura.md`; limites completos em `references/regras-seguranca.md`.

## Quando carregar cada referência

Todas em `references/`. Carregar sob demanda, não de antemão.

| Situação | Carregar |
|---|---|
| Há ferramenta/MCP/CLI capaz de enviar SQL | `contrato-operacional-claude.md` |
| Antes de qualquer diagnóstico contra produção | `politica-leitura-producao.md` |
| Configurar ou revisar o acesso do agente ao banco | `acesso-somente-leitura.md` |
| Revisar índices, planos ou reescrita de consulta | `indices-desempenho.md` |
| Propor `ALTER TABLE`, FK, mudança de PK ou backfill | `seguranca-ddl.md` |
| Produzir revisão formal escrita | `formato-saida.md` |
| Dúvida sobre um limite de segurança (G1–G17) | `regras-seguranca.md` |

## Verificação prévia de execução

Antes do primeiro comando SQL por MCP, CLI, IDE ou outro conector, verificar:

1. ambiente de destino: desenvolvimento, homologação, réplica ou produção;
2. versão do MySQL e esquema;
3. se a identidade conectada é a conta aprovada de análise somente leitura;
4. se a sessão permite execução de consultas ou apenas inspeção de metadados.

**Se a identidade ou o ambiente não puderem ser verificados, não executar SQL.**

## Fluxo principal

1. Identificar o problema e o resultado esperado.
2. Inventariar as evidências disponíveis: SQL, DDL da tabela, quantidade de linhas, índices, `EXPLAIN`, tempo de execução, frequência, versão do MySQL e contexto de produção.
3. Identificar evidências importantes ausentes. Não inventar esquema, cardinalidade, seletividade, tráfego ou uso de índices.
4. Analisar a consulta e o esquema atuais antes de propor mudanças.
5. Determinar o gargalo mais provável e separar fato de hipótese.
6. Avaliar o menor conjunto seguro de alternativas.
7. Classificar o risco operacional.
8. Recomendar a menor mudança justificável.
9. Definir validação antes/depois e reversão.
10. Gerar SQL somente quando a recomendação estiver suficientemente sustentada.

Se as evidências forem insuficientes para uma conclusão segura, informar exatamente o que falta e fornecer os comandos de diagnóstico necessários para coletar essas informações.

## Hierarquia de evidências

1. Métricas reais de execução e observabilidade de produção já coletadas.
2. `EXPLAIN ANALYZE` — **somente quando executar a consulta for comprovadamente seguro** no ambiente alvo. Ele executa a instrução; em produção, o padrão é não usar.
3. `EXPLAIN FORMAT=JSON` ou `EXPLAIN` — não executa a consulta; é o ponto de partida padrão em produção.
4. `SHOW CREATE TABLE` e definições dos índices existentes.
5. Quantidade de linhas, estimativas de cardinalidade/seletividade, frequência da consulta e distribuição dos dados.
6. Heurísticas gerais, somente quando nada melhor estiver disponível.

Nunca apresentar uma heurística como fato medido.

## Regras obrigatórias de análise

Detalhe completo em `references/indices-desempenho.md`.

- Inspecionar `SHOW CREATE TABLE`/`SHOW INDEX` antes de propor índice; verificar duplicata exata e redundância por prefixo à esquerda.
- Derivar a ordem das colunas de índice composto do caminho real de acesso — igualdade, intervalo, junção, ordenação/agrupamento, seletividade —, não de uma regra universal.
- Índice de cobertura só quando a carga justificar o armazenamento extra e a amplificação de escrita.
- Conferir tipos, charset/collation e signedness em colunas de junção e filtro; sinalizar conversão implícita.
- Avaliar o desenho da chave primária: o InnoDB clusteriza as linhas por ela e replica seu valor nas folhas de todo índice secundário. Preferir chave estreita, estável e crescente (`BIGINT`) quando o domínio permitir; UUID aleatório como PK é compensação que exige justificativa, não proibição.
- Rastrear os anti-padrões de consulta: `SELECT *`, predicado não sargável, função sobre coluna indexada, `LIKE '%x%'`, lista `IN` gigante, `OR`, paginação por `OFFSET` profundo, subconsulta correlacionada, resultado desnecessariamente grande.
- Não tratar varredura completa de tabela, `Using filesort` ou `Using temporary` como defeito automático: em tabela pequena ou predicado pouco seletivo podem ser a escolha certa. Julgar por custo medido, linhas processadas, loops, latência e frequência.
- Pesar o ganho de leitura contra custo de escrita, pressão no buffer pool, espaço em disco, redo/undo e manutenção de índices.

## Segurança de DDL

Tratar mudança de esquema como mudança operacional, não como sintaxe. Detalhe completo em `references/seguranca-ddl.md`.

Antes de recomendar `ALTER TABLE`, levantar: tamanho e quantidade de linhas da tabela, versão do MySQL, algoritmo viável (`INSTANT`/`INPLACE`/`COPY`), bloqueios e bloqueio de metadados, duração esperada versus tráfego, replicação, espaço em disco e reversão.

`ALGORITHM=INPLACE` não significa "sem bloqueio" nem "sem impacto". `LOCK=NONE` não elimina bloqueio de metadados. Mudança de chave primária e DDL em tabela grande ou quente são risco ALTO por padrão.

## Limites inegociáveis

- Nunca recomendar `DROP DATABASE`, `DROP TABLE`, `TRUNCATE`, remoção destrutiva de colunas ou exclusão em massa como otimização rotineira.
- Nunca gerar `UPDATE` ou `DELETE` sem escopo para uso em produção sem identificar explicitamente limite e salvaguardas.
- Nunca sugerir desabilitar verificação de chave estrangeira, binary logging, durabilidade ou proteções de integridade apenas para acelerar uma mudança sem análise explícita de risco.
- Nunca propor novo índice sem verificar os índices existentes quando essa informação puder ser obtida.
- Nunca remover índice apenas porque um plano de execução não o utilizou.
- Nunca afirmar ganho de desempenho sem plano de medição, nem inventar percentuais ou tempos.
- Nunca assumir comportamento de DDL específico de versão do MySQL sem verificar a versão quando isso afetar a segurança.
- Nunca expor credenciais, senhas, segredos ou DSN contendo segredo em prompts, logs, relatórios ou artefatos.
- Preferir mudanças mínimas e reversíveis a redesenhos amplos.

## Classificação de risco

- **BAIXO**: diagnósticos somente leitura ou mudanças com impacto operacional desprezível.
- **MEDIO**: mudanças de índice/esquema/consulta com impacto limitado, mas que exigem testes e planejamento de implantação.
- **ALTO**: DDL em tabela grande, modificação massiva de dados, mudança de chave primária, reestruturação de chave estrangeira, operações com chance de bloquear tráfego ou mudanças com reversão incerta.
- **CRITICO**: ações destrutivas ou irreversíveis, risco de perda de dados, quebra de integridade ou operação cujo impacto em produção não possa ser delimitado com as evidências disponíveis.

Para risco ALTO ou CRITICO, não apresentar SQL bruto como instrução casual de "rode isso". Colocar pré-requisitos, salvaguardas, validação e reversão antes do comando.

## Formato de saída

Usar esta ordem em revisões relevantes, omitindo seções realmente não aplicáveis:

- **Diagnóstico** — problema observado e causa mais provável, separando fato confirmado de hipótese.
- **Evidências** — nós relevantes do plano, definições de índices, linhas estimadas/reais, tempos, filtros, bloqueios e fatos de esquema que sustentem o diagnóstico.
- **Risco** — `BAIXO`, `MEDIO`, `ALTO` ou `CRITICO`, com o motivo.
- **Recomendação** — a menor mudança justificável primeiro; alternativas rejeitadas ou de menor prioridade quando útil.
- **SQL / diagnósticos** — somente depois de justificar a recomendação; diagnósticos antes de mutação quando as evidências estiverem incompletas.
- **Impacto esperado** — o que deve melhorar e o que pode piorar em contrapartida. Não inventar ganhos numéricos.
- **Validação** — medições antes/depois: plano de execução, latência, linhas examinadas, correção do resultado, impacto de escrita, bloqueios e carga representativa.
- **Reversão** — como reverter a mudança ou recuperar com segurança.

Para análise de produção, finalizar com um destes estados: `PRONTO_PARA_REVISAO`, `PRECISA_DE_MAIS_EVIDENCIAS`, `EVIDENCIA_SEGURA_INSUFICIENTE` ou `NENHUMA_MUDANCA_RECOMENDADA`.

## Comandos úteis de diagnóstico

```sql
SELECT VERSION();
SHOW CREATE TABLE schema_name.table_name;
SHOW INDEX FROM schema_name.table_name;
EXPLAIN FORMAT=JSON SELECT ...;
```

Usar `EXPLAIN ANALYZE` somente quando a execução for segura para a instrução e o ambiente de destino.

Para investigar uso de índices, utilizar visões relevantes do Performance Schema ou do esquema `sys` quando disponíveis, reconhecendo que contadores podem ser reiniciados e que a janela de carga importa.

## Padrão de comunicação

- Ser preciso sobre o que é conhecido e o que é inferido.
- Explicar por que um índice ou reescrita ajuda em termos de caminho de acesso.
- Rejeitar explicitamente índices redundantes.
- Sinalizar quando o banco provavelmente não é o gargalo.
- Preferir uma recomendação bem sustentada a várias mudanças especulativas.
- Se duas alternativas forem viáveis, comparar risco operacional, benefício de leitura, custo de escrita e manutenibilidade.
