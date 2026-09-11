# Regras de segurança MySQL/InnoDB

## G1 - Evidência antes de mudanças
Não propor mudanças estruturais como primeira reação a uma consulta lenta. Sempre que possível, inspecionar primeiro a consulta, o esquema, os índices existentes e o plano de execução.

## G2 - Sem índices redundantes
Não criar índice antes de verificar duplicatas exatas, redundância por prefixo à esquerda, sobreposição com índices únicos e finalidade na carga de trabalho atual.

## G3 - Sem remoção especulativa
Não remover índice porque ele não apareceu em um plano ou pareceu não utilizado em uma janela curta. Estabelecer evidência de carga, risco de dependências, cobertura duplicada e reversão.

## G4 - SQL destrutivo
Tratar `DROP DATABASE`, `DROP TABLE`, `TRUNCATE`, `ALTER` destrutivo e `DELETE` amplo como CRITICO, salvo quando a tarefa for explicitamente descomissionamento/migração controlada com proteções de recuperação.

## G5 - Escopo de mutação
Para `UPDATE`/`DELETE`, verificar predicado limitado e fornecer um `SELECT` prévio ou etapa de contagem/inspeção quando apropriado. Mutações de alto volume exigem orientação de processamento em lotes.

## G6 - Consciência de versão em DDL
Não afirmar que um ALTER é instantâneo, sem interrupção, sem bloqueio ou seguro sem conhecer o suficiente sobre versão do MySQL, tipo da operação e contexto da tabela.

## G7 - Consciência de execução do EXPLAIN ANALYZE
Lembrar que `EXPLAIN ANALYZE` executa a consulta. Não utilizá-lo casualmente para cargas de trabalho caras ou em contexto inseguro de produção. Nunca tratá-lo como simples validação do analisador sintático sem execução.

## G8 - Integridade e durabilidade
Não desabilitar casualmente validação de chave estrangeira, constraints únicas, binary logging, configurações de redo/durabilidade ou proteções transacionais por velocidade.

## G9 - Credenciais e escopo de produção
Exigir conta dedicada de menor privilégio e somente leitura para análise direta do banco, especialmente em produção. Não usar `root`, DBA, migração, usuário de aplicação com escrita ou outra credencial privilegiada quando uma conta de análise somente leitura puder atender à tarefa.

A conta de análise não deve possuir privilégios de mutação ou DDL como `INSERT`, `UPDATE`, `DELETE`, `CREATE`, `ALTER`, `DROP`, `TRIGGER`, `EVENT`, `FILE`, `SUPER`, `SYSTEM_USER`, `GRANT OPTION` ou privilégios administrativos equivalentes. Conceder somente leitura e acesso a metadados no escopo mínimo de esquema exigido pelo diagnóstico.

Nunca solicitar elevação temporária de privilégio para facilitar a análise. Nunca recorrer a credenciais de `root`, DBA, migração, implantação ou aplicação com escrita. Se a evidência necessária não puder ser obtida com a conta aprovada somente leitura, reportar a evidência ausente e pedir que um operador autorizado a colete.

Credenciais somente leitura são barreira contra mutação, não garantia de segurança de carga. `SELECT`, `EXPLAIN ANALYZE`, varreduras grandes, ordenações, agregações ou diagnósticos concorrentes ainda podem degradar produção. Aplicar regras de segurança de custo e execução independentemente das permissões da conta.

Nunca expor credenciais, senhas, segredos ou DSNs contendo segredos em prompts, logs, relatórios, exemplos SQL ou artefatos.

## G10 - Dados sensíveis
Não solicitar nem expor datasets completos de produção quando esquema, planos, amostras, agregados ou exemplos anonimizados forem suficientes. Evitar credenciais, segredos, dados pessoais ou tokens nos artefatos de análise.

## G11 - Desempenho medido
Nunca afirmar "X% mais rápido" ou melhoria específica de latência sem medições antes/depois que sustentem a afirmação. Caso contrário, descrever apenas o impacto direcional esperado.

## G12 - Correção da consulta primeiro
Uma otimização que altera a semântica não é aceitável, salvo quando a mudança for intencional e validada explicitamente.

## G13 - Minimalidade
Preferir reescrita de consulta, projeções menores, reutilização correta de índice ou um único índice justificado em vez de vários índices especulativos.

## G14 - Amplificação de escrita
Todo índice secundário possui custo de armazenamento e escrita. Em tabelas com muitos INSERT/UPDATE, considerar explicitamente esse custo antes de adicionar índices.

## G15 - Reversão
Toda mudança de produção com risco MEDIO/ALTO/CRITICO deve incluir estratégia realista de reversão ou recuperação. Se a reversão for complexa, informar isso antes do SQL.

## G16 - Incerteza
Quando contexto crítico estiver ausente, não declarar a mudança como segura. Solicitar ou gerar comandos para coletar a evidência faltante.

## G17 - Fronteira de execução Claude/ferramenta
Quando Claude for o agente que consome esta skill, tratar qualquer conector de banco, servidor MCP, CLI, comando shell, integração de IDE ou ferramenta customizada capaz de enviar SQL como fronteira de execução.

Antes de usar uma dessas ferramentas contra produção:
- verificar se a identidade MySQL configurada é a identidade aprovada de análise somente leitura;
- não confiar apenas em instruções de prompt para impedir escrita; impor a restrição por grants do MySQL;
- se a configuração da ferramenta expuser identidade privilegiada, não executar SQL com ela;
- não alterar configuração do conector, credenciais, grants ou privilégios da sessão para contornar a restrição;
- preferir comandos de inspeção que não executem a consulta de negócio;
- interromper e reportar quando a ferramenta disponível não puder garantir a identidade aprovada ou o ambiente de destino.

A skill é um controle comportamental adicional. O modelo de permissões do MySQL é a barreira primária contra mutações.
