# Acesso MySQL somente leitura para análise por IA

Usar uma conta MySQL dedicada para Claude/ferramentas de análise de banco. Impor o comportamento somente leitura por grants do MySQL, não apenas por instruções de prompt.

## Base recomendada

Preferir acesso por esquema e conceder somente o que o fluxo de diagnóstico realmente exigir. Não reutilizar credenciais de aplicação, migração, DBA, implantação ou `root`.

Um ponto de partida conceitual típico é:

```sql
CREATE USER 'ai_db_analysis'@'<host-aprovado>' IDENTIFIED BY '<segredo-gerenciado>';
GRANT SELECT ON application_schema.* TO 'ai_db_analysis'@'<host-aprovado>';
```

Ajustar acesso a metadados/observabilidade separadamente e de forma mínima quando necessário. Não copiar um conjunto amplo de grants de produção apenas porque uma view de diagnóstico não está disponível.

## Restrições

A identidade de análise não deve receber privilégios de mutação, DDL, gerenciamento de grants, arquivos ou administração. Em especial, evitar `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `ALTER`, `DROP`, `TRIGGER`, `EVENT`, `FILE`, `SUPER`, `SYSTEM_USER` e `GRANT OPTION`.

Não incorporar senhas na skill, repositório, prompt, histórico de shell, logs ou relatórios gerados. Usar o mecanismo de segredos aprovado pelo ambiente.

## Limitação importante

Somente leitura não significa ausência de risco. `SELECT`, agregações, ordenações, junções, varreduras e `EXPLAIN ANALYZE` podem consumir CPU, I/O, buffer pool, espaço temporário e recursos de concorrência. Aplicar regras de segurança de custo antes de executar diagnósticos em produção.
