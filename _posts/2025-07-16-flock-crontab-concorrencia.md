---
title: "Evite concorrência em scripts agendados com crontab usando flock"
date: 2025-07-16 16:00:00 -0300
categories: [linux, automacao, shell, crontab]
---

Quando agendamos um script com `crontab`, muitas vezes esquecemos de um detalhe importante: **o que acontece se o script ainda estiver rodando quando o próximo agendamento for disparado?**

Neste post, vamos explicar:

- Qual é o problema de concorrência com scripts no cron
- Como resolver isso com `flock`
- Como usar `-n` para evitar bloqueios
- Um exemplo real com agendamento a cada 5 minutos
- Links para aprofundamento
- Outras alternativas de solução

---

## O problema: concorrência indesejada

Vamos supor que você agende um script assim:

```cron
*/5 * * * * /home/ubuntu/meuscript.py
```

Se esse script demorar 6 minutos para rodar, ao chegar o próximo ciclo de 5 minutos, o `cron` **iniciaria uma nova instância do mesmo script**, mesmo que o anterior ainda esteja rodando. Isso pode causar:

- Conflito de acesso a arquivos (renomear, deletar, mover)
- Sobrecarga de CPU/memória
- Inconsistências em bancos de dados
- Logs corrompidos ou confusos

---

## A solução: `flock`

O comando `flock` é uma solução simples e eficiente para **garantir que apenas uma instância do seu script rode por vez**.

### Sintaxe:

```bash
flock -n /tmp/nome.lock comando_a_executar
```

- O arquivo `/tmp/nome.lock` serve apenas como controle (pode ser vazio)
- O `flock` impede que outro processo com o mesmo lock seja executado
- Ao final do script, o lock é liberado automaticamente

### Nosso exemplo real:

```cron
*/5 * * * * flock -n /tmp/baseiw.lock /home/ubuntu/base-iw/venv/bin/python /home/ubuntu/base-iw/import_baseiw.py >> /home/ubuntu/base-iw/import.log 2>&1
```

Isso garante que:
- O script só roda se não houver outra execução ativa
- Se o lock estiver ocupado, nada é executado
- Tudo é logado no arquivo `import.log`

---

## O que significa o `-n`?

O argumento `-n` significa **"non-blocking"**:
- Se o lock **não puder ser obtido**, o `flock` **falha imediatamente**
- Sem `-n`, o processo ficaria esperando o lock ser liberado (o que poderia atrasar ou bloquear outras tarefas)

---

## Referências

- [Documentação oficial do `flock`](https://man7.org/linux/man-pages/man1/flock.1.html)
- [Post no StackOverflow sobre `flock`](https://stackoverflow.com/questions/1517337/what-is-the-best-way-to-make-a-shell-script-wait-until-a-file-exists)
- [Explicação interativa: flock + crontab](https://unix.stackexchange.com/questions/26583/how-can-i-prevent-cron-from-running-my-job-multiple-times-if-it-takes-a-long-tim)

---

## Outras alternativas

Se você quiser mais controle ou usar alternativas ao cron, considere:

- **Controle por `systemd`**: crie um `.service` com `Restart=on-failure` e `StartLimitIntervalSec`
- **Lock manual com shell script**: usando arquivos temporários (`touch`, `rm`) e validação de PID
- **Controle dentro do próprio script**: com `ps`, `pgrep` ou `lockfile`
- **Fila de processamento**: via Redis, RabbitMQ ou banco de dados

---

## Conclusão

Usar `flock` é uma das formas mais simples e seguras de evitar que seus scripts agendados via `crontab` rodem em paralelo. Com uma simples linha de cron e um lockfile temporário, você ganha robustez sem precisar mudar o seu fluxo de execução.

> Quer ver mais dicas como essa? Acompanhe os posts em [hlgurgel/blog](https://github.com/hlgurgel/blog).
