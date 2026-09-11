# Formato de revisão formal do banco

Usar este modelo quando o usuário pedir revisão formal ou quando a mudança afetar produção.

## Diagnóstico
- **Observado:** fatos sustentados diretamente por SQL/esquema/plano/métricas.
- **Hipótese:** causas prováveis ainda não confirmadas.

## Evidências
Incluir somente evidências relevantes, como chave escolhida, tipo de acesso, linhas examinadas, loops, latência, tamanho da tabela, índices, cardinalidade/seletividade e capacidade de DDL.

## Risco
**Nível:** BAIXO | MEDIO | ALTO | CRITICO

Explicar risco operacional e de integridade dos dados em linguagem clara.

## Recomendação
Apresentar primeiro a menor mudança preferida e explicar por que ela é melhor.

Se houver alternativas, comparar:

| Alternativa | Benefício de leitura | Custo de escrita/armazenamento | Risco operacional | Recomendação |
|---|---|---|---|---|

## SQL / diagnósticos
Colocar diagnósticos antes de mutação/DDL quando ainda forem necessárias evidências adicionais.

Para SQL potencialmente perigoso, adicionar pré-requisitos explícitos e não apresentá-lo como comando incondicional para execução.

## Impacto esperado
Descrever melhoria direcional e trade-offs. Não inventar percentuais nem tempos.

## Validação
Especificar:

1. consulta/plano e métricas de referência inicial;
2. verificação de correção;
3. consulta/plano e métricas após mudança;
4. observações do caminho de escrita/bloqueios/replicação quando relevantes;
5. critérios de aceite.

## Reversão
Informar como reverter, se a própria reversão pode bloquear ou ser cara e quais evidências devem disparar a reversão.
