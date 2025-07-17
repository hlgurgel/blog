---
title: "Importação eficiente de arquivos gigantescos no Python com pyodbc, pandas e logging"
date: 2025-07-16 17:00:00 -0300
categories: [python, banco-de-dados, automacao, data-engineering]
---

Neste post, vamos compartilhar a solução que desenvolvemos para lidar com a **importação automática de arquivos `.xlsx` e `.csv` com milhões de linhas** em uma base SQL Server, usando **Python**.

---

## 🎯 O problema

Precisávamos importar arquivos fornecidos por terceiros com até 1 milhão de linhas (ou mais), contendo informações estruturadas como datas, identificadores, e nomes de pontos de venda. Os arquivos podiam estar em:

- `.csv`
- `.xls`
- `.xlsx`

Além do volume massivo de dados, encontramos os seguintes desafios:

- **Formatos de datas inconsistentes**
- **Linhas incompletas ou inválidas**
- **Possibilidade de duplicatas**
- **Necessidade de performance**
- **Agendamento recorrente e seguro**

---

## 📅 Desafio: campo de data com múltiplos formatos

Os arquivos chegavam com datas em padrões mistos, como:

- `06/05/2025` (dd/MM/yyyy)
- `6/28/25` (MM/dd/yy)
- `2025-06-28` (ISO)

Para isso usamos:

```python
df_valid['DATA_ATIVACAO'] = pd.to_datetime(df_valid['DATA_ATIVACAO'], dayfirst=True, errors='coerce')
```

O parâmetro `dayfirst=True` nos protege contra interpretações erradas em arquivos vindos do Brasil. O `errors='coerce'` transforma erros de conversão em `NaT`, permitindo a filtragem das linhas inválidas depois.

---

## 🧹 Detecção de linhas inválidas

Validamos cada linha verificando se **todas as colunas obrigatórias** estão presentes e preenchidas. Se algo estiver vazio, registramos a linha como inválida no log com o motivo.

Isso garante que:
- As linhas que entram no banco estão corretas
- Arquivos corrompidos ou mal formatados não travam o processo

---

## ⚙️ pyodbc vs pandas.to_sql: por que escolhemos o pyodbc?

Embora o `pandas.to_sql()` seja mais simples, **ele não respeita a cláusula `IGNORE_DUP_KEY` do SQL Server** — o que significa que ele falha ao tentar inserir duplicatas.

Com o `pyodbc`, conseguimos usar:

```python
cursor.fast_executemany = True
cursor.executemany(insert_sql, insert_data)
```

E o SQL Server ignora as duplicatas silenciosamente, mantendo a integridade da tabela.

---

## 🚀 Uso do `fast_executemany` para alta performance

O `fast_executemany = True` é uma configuração do `pyodbc` que permite enviar grandes volumes de dados de forma vetorizada, evitando múltiplas roundtrips entre Python e o SQL Server.

Sem ele, inserir 1 milhão de linhas pode levar horas.
Com ele, o mesmo processo leva minutos.

---

## 📦 Divisão em lotes: controle e feedback

Mesmo com `executemany`, dividimos a inserção em **lotes de 10.000 linhas**, assim:

- O banco de dados não trava com um bloco gigante
- Podemos mostrar no log o progresso para o usuário
- Facilitamos o diagnóstico em caso de falha (sabemos em qual lote parou)

Exemplo:

```python
for i in range(total_batches):
    logger.info(f"Inserindo lote {i + 1}/{total_batches}...")
    cursor.executemany(insert_sql, batch)
```

---

## 📓 Logging estruturado com `logging`

Usar o módulo `logging` do Python foi essencial para:

- Criar arquivos `.log` por arquivo processado
- Registrar erros com detalhes
- Acompanhar o progresso com `watch -n 10`

Exemplo:

```python
logging.basicConfig(filename='arquivo.log', level=logging.INFO)
logger.info("Iniciando processamento do arquivo X...")
```

---

## 🕒 Agendamento com segurança: `crontab + flock`

Agendamos o script para rodar a cada 5 minutos com `crontab`, mas usamos `flock` para evitar execuções simultâneas:

```cron
*/5 * * * * flock -n /tmp/baseiw.lock /home/ubuntu/base-iw/venv/bin/python /home/ubuntu/base-iw/import_baseiw.py
```

🔗 Veja nosso post completo sobre isso: [Evite concorrência em scripts agendados com crontab usando flock](/linux/automacao/shell/crontab/2025/07/16/flock-crontab-concorrencia.html)

---

## 🔗 Referências úteis

- [pandas.to_datetime](https://pandas.pydata.org/pandas-docs/stable/reference/api/pandas.to_datetime.html)
- [pyodbc docs](https://github.com/mkleehammer/pyodbc/wiki)
- [fast_executemany explicado](https://github.com/mkleehammer/pyodbc/wiki/fast_executemany)
- [logging — Logging facility for Python](https://docs.python.org/3/library/logging.html)
- [Post anterior sobre `flock` e `crontab`](https://github.com/hlgurgel/blog/blob/main/_posts/2025-07-16-flock-crontab-concorrencia.md)

---

## ✅ Conclusão

Com essa arquitetura, conseguimos:

- Processar arquivos gigantescos com robustez
- Evitar erros de data e de duplicidade
- Garantir rastreabilidade com logs
- Escalar o processo com segurança

Ficou com dúvidas ou quer adaptar isso para sua realidade? Comente lá no [repositório do blog](https://github.com/hlgurgel/blog) ou me chame por aqui.
