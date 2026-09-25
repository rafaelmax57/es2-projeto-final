# Plano de testes - Parte 1

## Módulo escolhido

Escolhi o `flaskbb/forum/`, e dentro dele o foco vai ser o `models.py`.

O forum é a parte principal do sistema (categorias, fóruns, tópicos e posts), então faz sentido testar ele. O `management` tem a menor cobertura (7%), mas quase tudo lá é view e form de administração, que precisa de request, login de admin etc. No forum, o `models.py` tem bastante lógica que dá pra testar direto chamando os métodos, sem precisar subir página. O `views.py` e o `forms.py` do forum também estão quase sem cobertura, mas deixei eles de fora por enquanto pelo mesmo motivo do management.

## Cobertura atual (baseline)

| Arquivo | Linhas | Linhas sem cobertura | Branches | Branches parciais | Cobertura |
|---|---|---|---|---|---|
| `flaskbb/forum/` (total) | 1272 | 854 | 284 | 36 | 34% |
| `flaskbb/forum/models.py` | 641 | 271 | 150 | 33 | 60% |

Esses números são do `BASELINE.md`, com o comando `--cov-branch`.

## Meta

- Subir o `forum` de 34% pra pelo menos 35% (+1 pp).
- Subir o `forum/models.py` de 60% pra pelo menos 62% (+2 pp).

A meta parece pequena, mas o forum tem 1272 linhas e mais da metade das que faltam estão no `views.py` (465) e no `forms.py` (91), que eu não vou atacar nessa parte. Então cada linha nova coberta no `models.py` muda pouco na porcentagem do módulo inteiro.

## Cenários que ainda não estão cobertos

| # | Cenário | Tipo |
|---|---|---|
| 1 | Criar um `Topic` passando o usuário (tem que preencher `user_id` e `username`) | feliz |
| 2 | Criar um `Topic` passando o `content` (tem que criar o primeiro post) | feliz |
| 3 | Criar um `Topic` sem nenhum argumento (não pode dar erro) | borda |
| 4 | `Topic.is_first_post` com o primeiro post e com um post novo | feliz / borda |
| 5 | `Topic.get_topic` com id que existe | feliz |
| 6 | `Topic.get_topic` com id que não existe (tem que dar 404) | erro |
| 7 | `Forum.get_forum` com id que não existe, logado e deslogado (404) | erro |
| 8 | `Topic.slug` com vários títulos, incluindo título só com pontuação | feliz / borda |
| 9 | `Topic.tracker_needs_update` com tópico antigo e recente (depende da data atual) | feliz / borda |
| 10 | `Topic.first_unread` devolvendo o primeiro post não lido | feliz |
| 11 | `Topic.get_topic` com `hiddencheck=True` num tópico escondido | borda |

O item 8 vai ser o teste parametrizado (tarefa 1.4) e o item 9 o teste com mock (tarefa 1.5), porque ele usa o `time_utcnow()`.
