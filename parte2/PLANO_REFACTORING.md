# Plano de refactoring - Parte 2

Escolhi 4 smells do `CODE_SMELLS.md` pra tratar agora. A ideia é fazer uma refatoração por commit e rodar a suíte inteira depois de cada uma.

## R1 - Long Method em `Forum.update_read` (smell 1)

- **Refatoração:** Extract Method. Tirar a consulta que conta os tópicos não lidos pra um método novo, `Forum._count_unread_topics(user, read_cutoff)`.
- **Resultado esperado:** o `update_read` fica mais curto e dá pra ler a regra de marcar o fórum como lido sem passar pela query.
- **Riscos:** a query usa `user` e `read_cutoff`, então se eu esquecer de passar algum dos dois o resultado muda. O método é chamado pelo `Topic.update_read` e pelas views, mas como a assinatura do `update_read` não muda, quem chama não é afetado.

## R2 - Duplicated Code no cálculo do `read_cutoff` (smell 2)

- **Refatoração:** Extract Function. Criar uma função `_get_read_cutoff()` no próprio `models.py` e usar ela no `Topic.tracker_needs_update` e no `Forum.update_read`.
- **Resultado esperado:** a regra do tempo do tracker fica num lugar só.
- **Riscos:** o teste com mock da Parte 1 troca o `flaskbb.forum.models.time_utcnow`. Se a função nova ficar em outro módulo, o mock para de funcionar. Por isso ela vai ficar no mesmo arquivo. Também tem que continuar voltando `None` quando o `TRACKER_LENGTH` for 0.

## R3 - Duplicated Code ao zerar o último post do fórum (smell 3)

- **Refatoração:** Extract Method. Criar `Forum.clear_last_post()` e usar ele nos três lugares que zeram os 5 campos.
- **Resultado esperado:** as 15 linhas repetidas viram 3 chamadas de um método só.
- **Riscos:** dois dos três lugares estão em outras classes (`Post` e `Topic`) e acessam o fórum por `self.topic.forum` e `self.forum`. Tenho que chamar o método no objeto certo. Também não posso mudar o `last_post_user` pra `last_post_user_id`, porque no código original está `last_post_user`.

## R4 - Dead Code em `Topic.update_read` (smell 4)

- **Refatoração:** Remove Dead Code, junto com Remove Dead Assignment na variável `updated`.
- **Resultado esperado:** o método fica só com os dois casos que existem (atualizar ou criar o `TopicsRead`) e retorna direto o resultado do `forum.update_read`.
- **Riscos:** o retorno tem que continuar sendo exatamente o do `forum.update_read`, porque é isso que o código faz hoje (o `updated` dos `if` é sempre sobrescrito). Se eu retornar `True` sem querer, muda o comportamento.

## Smells que ficam pra depois

- **Data Clumps (5) e Feature Envy (6):** o certo seria criar algo como um método `Forum.set_last_post(...)` e mover a lógica do `_deal_with_last_post` pro `Forum`. Isso mexe em muitos lugares de uma vez, então preferi não fazer agora. O R3 já é um primeiro passo nessa direção.
- **Comentário desatualizado (7) e nome enganoso (8):** vão entrar nas melhorias de legibilidade (tarefa 2.4).
