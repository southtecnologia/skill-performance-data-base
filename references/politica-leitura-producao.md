# Política de leitura em produção

Usar esta política sempre que Claude ou outro agente de IA puder executar SQL diretamente em ambiente MySQL/InnoDB de produção.

## Postura padrão

Tratar produção com prioridade para observação. Preferir metadados e diagnósticos passivos antes de executar consultas de negócio.

## Diagnósticos preferenciais de baixo impacto

Usar primeiro, quando disponíveis e autorizados:

- `SELECT VERSION()`;
- `SHOW CREATE TABLE ...`;
- `SHOW INDEX FROM ...`;
- `EXPLAIN ...` sem executar a consulta subjacente;
- metadados de esquema em `information_schema`;
- observações existentes do Performance Schema ou `sys` que já estejam sendo coletadas;
- agregações limitadas ou amostras somente quando o caminho de acesso for conhecido.

Baixo impacto não é impacto zero. Consultas amplas a `information_schema.TABLES` ou `information_schema.COLUMNS` podem ser caras em servidores com muitas tabelas, porque algumas colunas exigem abrir os arquivos de tabela. Filtrar sempre por `TABLE_SCHEMA`/`TABLE_NAME` e evitar varrer o catálogo inteiro.

## Comandos que exigem cuidado adicional

Não executar automaticamente apenas porque a conta é somente leitura:

- `EXPLAIN ANALYZE` em consulta cara ou de custo incerto;
- `SELECT` amplo sem predicados seletivos;
- `SELECT *` em tabelas grandes;
- junções grandes, ordenações, agrupamentos, window functions, CTEs recursivas ou subconsultas com cardinalidade desconhecida;
- consultas contendo `LIKE '%...%'` em tabelas de alto volume;
- paginação profunda com `OFFSET`;
- contagens ou varreduras completas de tabelas grandes e quentes;
- diagnósticos com possibilidade de alocar temporary tables grandes;
- loops repetidos de benchmark contra produção.

Quando o custo não puder ser delimitado, interromper e solicitar fonte de evidência mais segura, ambiente de homologação representativo, réplica destinada a análise/diagnóstico ou plano/métricas fornecidos por operador.

## Segurança da sessão

Quando o ambiente suportar, preferir sessões com limites e controles operacionais em vez de depender apenas de intenção humana. Exemplos incluem timeout de execução no servidor, pool de conexão dedicado e concorrência restrita para a identidade de análise.

Não alterar variáveis globais do MySQL, configuração do servidor, defaults de isolamento, optimizer switches ou configurações de recursos para facilitar a análise.

## Segurança do volume de resultados

Minimizar linhas retornadas ao agente. Preferir projeções, agregados, metadados e amostras direcionadas. Não buscar dados sensíveis ou de alto volume em produção quando planos, contagens, distribuições ou amostras anonimizadas forem suficientes.

## Condições de parada

Interromper execução direta em produção e informar o motivo quando:

- a identidade conectada não puder ser verificada como usuário aprovado somente leitura;
- o host/ambiente de destino não puder ser verificado;
- o plano da consulta ou tamanho da tabela indicar trabalho sem limite;
- a análise necessária exigir privilégios de escrita/DDL/administração;
- a ferramenta ocultar o SQL real que será enviado;
- o diagnóstico puder competir de forma relevante com o tráfego de produção;
- o custo esperado não puder ser estimado com as evidências disponíveis.

O resultado correto pode ser `EVIDENCIA_SEGURA_INSUFICIENTE`. Não forçar conclusão quando obter evidência mais forte criaria risco inaceitável para produção.
