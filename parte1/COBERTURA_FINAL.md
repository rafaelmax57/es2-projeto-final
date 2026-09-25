# Cobertura final - Parte 1

## Comando

Usei o mesmo comando da baseline:

```powershell
uv run pytest --cov=flaskbb.forum --cov=flaskbb.management --cov=flaskbb.user --cov-branch --cov-report=term-missing
uv run coverage report --include="flaskbb/forum/*" --show-missing
```

Resultado: 250 passed.

## Saída do módulo forum

```text
Name                        Stmts   Miss Branch BrPart  Cover   Missing
-----------------------------------------------------------------------
flaskbb\forum\__init__.py       2      2      0      0     0%   1-3
flaskbb\forum\forms.py         91     91     16      0     0%   12-212
flaskbb\forum\locals.py        29     18      8      2    46%   11-23, 27-28, 34-35, 41-42, 44, 48, 55-58
flaskbb\forum\models.py       641    257    150     28    63%   12-201, 219-256, 268->271, 282-294, 312->exit, 320->337, 342-343, 347-348, 358-359, 361, 373-374, 376, 388, 391->421, 414-418, 423, 432, 454, 479, 491->exit, 497->exit, 508-575, 582-583, 587-588, 591, 594, 620-627, 634-663, 666, 673-688, 734, 784, 791-800, 828-829, 852-853, 868->872, 884-885, 898->901, 904-905, 911, 922, 925, 936, 951-957, 965, 969, 991, 1016, 1017->exit, 1028, 1035-1037, 1053-1123, 1127-1128, 1131, 1134-1146, 1177, 1202->1208, 1275, 1305-1306, 1317->1326, 1331-1332, 1354-1367, 1402-1403, 1449-1488, 1525-1526, 1594-1595, 1663
flaskbb\forum\utils.py         10      7      2      1    33%   12-25, 30->exit, 34
flaskbb\forum\views.py        499    465    108      0     6%   13-1165
-----------------------------------------------------------------------
TOTAL                        1272    840    284     31    35%
```

## Comparação

| | Baseline | Final | Diferença |
|---|---|---|---|
| `forum` (total) | 34% | 35% | +1 pp |
| linhas sem cobertura no forum | 854 | 840 | -14 |
| branches parciais no forum | 36 | 31 | -5 |
| `forum/models.py` | 60% | 63% | +3 pp |
| linhas sem cobertura no models.py | 271 | 257 | -14 |

Os outros módulos não mudaram (`management` 7% e `user` 34%). O total dos três foi de 25% pra 26%.

A meta do `PLANO_TESTES.md` era +1 pp no forum e +2 pp no `models.py`. As duas foram batidas: deu +1 pp no forum e +3 pp no `models.py`.

Comparando o "Missing" da baseline com o de agora, saíram da lista por exemplo o `605->608`, `612-613` e `616` (o `__init__` do `Topic`) e o `1388` e `1397` (os dois `abort(404)` do `get_forum`).

## O que continua sem cobertura

| Cenário | Onde | Como daria pra cobrir |
|---|---|---|
| `Topic.first_unread` | models.py 634-663 | criar um tópico com alguns posts, um `TopicsRead` com `last_read` no meio deles e ver se volta a URL do primeiro post depois dessa data |
| `Topic.get_topic` com `hiddencheck=True` | models.py 666 | esconder um tópico com `topic.hide(user)` e chamar `get_topic(id, hiddencheck=True)` esperando 404 |
| `Topic.get_posts` (paginação) | models.py 673-688 | criar mais posts que o `POSTS_PER_PAGE` e conferir quantos vêm em cada página |
| `Topic.hide` / `unhide` quando já está escondido ou visível | models.py 911 e 925 | chamar `hide` duas vezes seguidas e ver se a segunda não muda nada (mesma coisa com `unhide` num tópico visível) |
| `Topic.update_read` quando não tem post novo | models.py 784 | marcar o tópico como lido e chamar `update_read` de novo sem criar post, esperando que o tracker não seja atualizado |
| `Forum.move_topics_to` | models.py 1360-1363 | passar uma lista com dois tópicos e ver se os dois mudam de fórum |
| `Category.get_forums` com categoria que não existe | models.py 1663 | chamar com um id que não existe esperando 404, igual fiz com o `get_forum` |
| `forum/views.py` | 6% | testes com o `client` do Flask fazendo request nas páginas (ver tópico, criar post etc.) |
| `forum/forms.py` | 0% | instanciar os forms com dados válidos e inválidos dentro de um request de teste e chamar `validate()` |

## Observação

Algumas linhas continuam aparecendo como não cobertas mesmo sendo executadas, tipo `12-201` e `508-575` no `models.py` (imports e definição das classes) e várias linhas de `def` e `@classmethod`. Como falei no `BASELINE.md`, acho que é porque esse código roda no import, antes do coverage começar a medir. Por isso usei exatamente o mesmo comando nas duas medições, pra comparação ser justa.
