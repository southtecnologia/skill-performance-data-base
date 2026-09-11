# Segurança de DDL para MySQL/InnoDB

## Verificação prévia

Coletar ou solicitar:

```sql
SELECT VERSION();
SHOW CREATE TABLE schema_name.table_name;
SHOW INDEX FROM schema_name.table_name;
```

Também levantar tamanho aproximado da tabela, quantidade de linhas, taxa de escrita, janelas críticas de tráfego, topologia de replicação quando relevante e espaço livre em disco.

## Análise de ALTER TABLE

Para cada alteração proposta, determinar se a versão alvo do MySQL pode executá-la usando `INSTANT`, `INPLACE` ou se exige `COPY`. O comportamento varia conforme versão e tipo de alteração; não generalizar.

Analisar explicitamente:

- aquisição de bloqueio de metadados e possíveis bloqueadores;
- necessidade de rebuild/cópia;
- leituras/escritas concorrentes permitidas;
- consumo temporário de disco;
- pressão em redo/undo/binlog/replicação;
- incerteza de duração;
- comportamento em caso de falha e reversão.

`LOCK=NONE` não garante ausência total de bloqueio. Bloqueios de metadados ainda podem atrasar ou bloquear o DDL.

`ALGORITHM=INPLACE` não significa execução instantânea e ainda pode realizar trabalho significativo.

## Tabelas grandes

Tratar DDL em tabelas grandes ou de alto tráfego como risco ALTO, salvo quando evidências fortes delimitarem o impacto.

Considerar:

- DDL nativo sem interrupção quando a operação/versão exata o suportar com segurança;
- janela de manutenção;
- ferramenta/processo de alteração sem interrupção de esquema aprovado pela organização;
- migrações em fases e padrões de leitura/escrita dupla para mudanças arquiteturais.

Não prescrever ferramenta de terceiros para alteração sem interrupção de esquema sem confirmar que o ambiente a suporta e autoriza.

## Mudanças de chave primária

Alterar a chave primária pode reconstruir a tabela clusterizada e todos os índices secundários. Tratar como risco ALTO por padrão e exigir análise de dependências, armazenamento, duração, replicação e reversão.

## Chaves estrangeiras

Antes de adicionar ou alterar uma chave estrangeira:

- verificar compatibilidade de tipos de dados, tamanho, charset/collation e signedness, quando aplicável;
- verificar índices de suporte;
- detectar dados órfãos;
- considerar custo de validação e bloqueio;
- avaliar comportamento de cascade e expectativas da aplicação.

Não recomendar `SET FOREIGN_KEY_CHECKS=0` como atalho genérico de migração.

## Backfills de dados

Para operações grandes de UPDATE/preenchimento retroativo:

- limitar cada lote;
- usar range determinístico de chave ou outra estratégia segura de processamento em lotes;
- definir tamanho de transação;
- considerar lag de replicação, histórico de undo, bloqueios e contenção com a aplicação;
- tornar a operação retomável/idempotente quando prático;
- definir critérios de verificação e parada.

Nunca gerar preenchimento retroativo de produção sem limite apenas porque o SQL é sintaticamente válido.
