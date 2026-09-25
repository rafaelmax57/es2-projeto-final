# Novos testes - Parte 1

Todos os testes estão no mesmo arquivo do meu fork:

`tests/unit/forum/test_forum_models_extra.py`

https://github.com/rafaelmax57/flaskbb/blob/master/tests/unit/forum/test_forum_models_extra.py

Para rodar só eles:

```powershell
uv run pytest tests/unit/forum/test_forum_models_extra.py
```

E a suíte inteira continua passando com `uv run pytest` (250 passed, eram 233 na baseline).

## Tarefa 1.3 - casos novos

| Teste | Linha | Tipo | O que verifica |
|---|---|---|---|
| `test_topic_init_com_user` | 11 | feliz | criar tópico com usuário preenche `user_id` e `username` |
| `test_topic_init_com_content` | 18 | feliz | criar tópico com `content` já cria o primeiro post |
| `test_topic_init_sem_argumentos` | 25 | borda | `Topic()` sem nada não quebra e as duas datas começam iguais |
| `test_is_first_post_verdadeiro` | 32 | feliz | o primeiro post do tópico é reconhecido como first post |
| `test_is_first_post_falso` | 36 | borda | um post novo (sem id ainda) não é o first post |
| `test_get_topic_existente` | 43 | feliz | `get_topic` devolve o tópico quando o id existe |
| `test_get_topic_inexistente_da_404` | 47 | erro | `get_topic` dá 404 com id que não existe |
| `test_get_forum_inexistente_da_404_logado` | 52 | erro | `get_forum` dá 404 com id que não existe (usuário logado) |
| `test_get_forum_inexistente_da_404_guest` | 57 | erro | `get_forum` dá 404 com id que não existe (visitante) |

Fiz os dois do `get_forum` separados porque o método tem um caminho pra usuário logado e outro pra visitante, e cada um tem o seu `abort(404)`.

## Tarefa 1.4 - teste parametrizado

`test_topic_slug_parametrizado`, linha 75 (o `@pytest.mark.parametrize` começa na linha 64).

Testa a propriedade `Topic.slug` com 6 títulos:

| Título | Slug esperado | Tipo |
|---|---|---|
| `"Ola Mundo"` | `ola-mundo` | válido |
| `"Ação é legal"` | `acao-e-legal` | válido (tira acento) |
| `"  Muitos   espacos  "` | `muitos-espacos` | válido (espaços sobrando) |
| `"Topico 123"` | `topico-123` | válido (com número) |
| `"!!!"` | `""` | inválido |
| `"???"` | `""` | inválido |

Os dois últimos são títulos só com pontuação. O slug fica vazio, e isso é o que faz o `Topic.url` montar a URL sem o slug.

## Tarefa 1.5 - teste com mock

`test_tracker_needs_update_topico_antigo` (linha 83) e `test_tracker_needs_update_topico_recente` (linha 95).

Os dois usam o `mocker.patch` do pytest-mock pra trocar o `time_utcnow` dentro de `flaskbb.forum.models` por um mock que devolve uma data que eu escolho. No tópico antigo o "agora" é 30 dias depois do último post e o resultado tem que ser `False`. No recente é 1 hora depois e tem que ser `True`. Além do resultado, os dois conferem com `assert_called_once_with()` que o método chamou o relógio uma vez.

**Por que precisei do mock:** o `tracker_needs_update` compara a data do último post com a hora atual, e no fixture o post é criado na hora que o teste roda. Sem trocar o relógio não tem como simular um tópico antigo, a não ser esperando dias de verdade. Com o mock o teste fica controlado e dá sempre o mesmo resultado.

## Ajuste no fixture `database`

Quando comecei a adicionar testes, a suíte em paralelo começou a falhar de vez em quando. Descobri que o `Setting.as_dict()` fica em cache e esse cache não é apagado pelo `drop_all()`, então um teste que rodava sem settings deixava o cache errado pro próximo teste no mesmo processo.

Coloquei um `cache.clear()` no final do fixture `database` em `tests/fixtures/app.py`. Não mudei nenhum teste que já existia. Depois disso rodei a suíte completa 8 vezes seguidas e passou em todas.
