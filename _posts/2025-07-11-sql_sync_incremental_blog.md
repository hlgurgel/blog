---
layout: post
title: "Sincronização incremental de dados com múltiplas fontes no SQL Server"
date: 2025-07-11
author: Helder Jr.
---

> **Nota**: Todas as consultas SQL apresentadas neste artigo foram adaptadas para preservar a estrutura e segurança dos dados reais utilizados no projeto.

---

Esse é o primeiro artigo que publico por aqui, e a ideia é justamente essa: transformar soluções reais do dia a dia em material público, tanto para documentar o processo quanto para ajudar quem estiver enfrentando desafios parecidos.

Vamos ao caso.

## O desafio

Imagine que você tenha duas fontes de dados independentes que registram eventos sobre os mesmos objetos (no nosso caso, ICCIDs de chips de operadora). Ambas as fontes são válidas e se complementam, podendo gerar registros simultâneos ou intercalados para o mesmo ICCID.

A ideia era unificar essas informações em uma tabela central (`ICCID_Lifecycle`), garantindo que:

- Os dados fossem sincronizados automaticamente, de forma incremental;
- Nenhum evento fosse perdido (mesmo que tivesse delay na origem);
- A ordenação seguisse a cronologia real dos eventos;
- O processo fosse executado **de hora em hora**, com segurança e performance.

## Primeiras escolhas: estrutura da tabela e índices

Começamos com uma tabela que tinha um campo `id` do tipo `IDENTITY`, que era usado para ordenação. Porém, com fontes diferentes inserindo dados simultaneamente, isso passou a ser um problema: não era um identificador universal de tempo.

A solução foi:

- **Remover a coluna `id`** e deixar que a ordenação fosse feita por um campo real de data: `DataProcessamento`;
- **Alterar o tipo da coluna para `datetime2(3)`**, para evitar colisões por registros no mesmo segundo;
- **Criar um índice não clusterizado** em `ICCID` + `DataProcessamento` para garantir performance:

```sql
CREATE NONCLUSTERED INDEX ix_ICCID_DataProcessamento
ON dbo.ICCID_Lifecycle (ICCID, DataProcessamento)
INCLUDE (...);
```

## Atualização incremental com janela segura

A grande sacada foi adotar o conceito de **janela de atualização com atraso de segurança**. Isso significa que, ao invés de sincronizar os dados até o exato momento atual, aplicamos um delay de 1 hora.

Assim, garantimos que:

- Se algum evento for registrado com atraso na origem, ele ainda será capturado na próxima execução;
- Evitamos duplicidade reprocessando sempre a mesma janela, com exclusão + inserção;

### Fluxo do processo

1. Buscar a última data processada localmente (armazenada em uma tabela de controle);
2. Definir a janela de integração: entre essa data e a hora atual -1h;
3. Excluir os dados dessa janela na tabela local (o que, em processamento normal, nenhum registro será deletado.);
4. Inserir os novos dados das duas fontes usando essa janela;
5. Atualizar a tabela de controle com a maior `DataProcessamento` carregada.

## Exemplo de consulta adaptada (fonte estrutural)

```sql
SELECT iccid, data_evento, ...
FROM OPENQUERY(FONTE_REMOTA, '
  SELECT ...
  FROM base_eventos
  WHERE data_evento BETWEEN ''2025-07-10 10:00:00'' AND ''2025-07-10 11:00:00''
');
```

## Exemplo de consulta adaptada (fonte promotoria)

```sql
SELECT serial AS iccid, dataprocessamento, ...
FROM dbo.movimentacoes
WHERE dataprocessamento BETWEEN @inicio AND @fim;
```

## Resultado

- O processo leva **menos de 3 minutos** para rodar a cada hora;
- Não temos buracos nem duplicidade de dados;
- A ordenação cronológica está garantida;
- A estrutura está preparada para escalar ou ser auditada.

## Conclusão

Não foi preciso reinventar a roda. A solução veio da observação de um padrão simples: dados podem atrasar. Com isso, um pequeno "delay técnico" de segurança evitou a necessidade de estruturas complexas.

Essa é a primeira de muitas publicações que pretendo escrever aqui. Se você curtiu, fique de olho que vem mais soluções do campo de batalha por aí. ;)

---

[Voltar ao índice](/blog/)