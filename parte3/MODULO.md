# Módulo forum

## Propósito

O `flaskbb/forum/` é o coração do flaskbb. É ele que cuida da estrutura do fórum (categorias, fóruns, tópicos e posts) e do que o usuário faz no dia a dia: ver a lista de fóruns, abrir um tópico, criar tópico, responder, editar, apagar, esconder, denunciar post e marcar como lido. Ele também guarda os contadores (quantos posts e tópicos cada fórum tem), qual foi o último post de cada fórum e o controle do que cada usuário já leu (o "tracker" de leitura).

## Mapa dos arquivos

| Arquivo | Responsabilidade |
|---|---|
| `models.py` | Modelos do banco (`Category`, `Forum`, `Topic`, `Post`, `Report`, `TopicsRead`, `ForumsRead`) e as regras de salvar, apagar, esconder, mover e marcar como lido. |
| `views.py` | Rotas do blueprint `forum`, cada tela é uma `MethodView` (índice, fórum, tópico, novo post, editar, moderação etc.). |
| `forms.py` | Formulários WTForms de criar tópico, responder, editar post, denunciar e busca. |
| `locals.py` | `current_forum`, `current_topic`, `current_post` e `current_category`, que descobrem o item atual pelos parâmetros da URL. |
| `utils.py` | Verifica se precisa forçar login quando o fórum não aceita visitante. |
| `__init__.py` | Só cria o logger do módulo. |

## Entradas e saídas

### Entradas

- Rotas HTTP: o blueprint `forum` é registrado pelo hook `flaskbb_load_blueprints` (`views.py:1165`) com o prefixo `FORUM_URL_PREFIX`. As principais são `/` (índice), `/category/<id>`, `/forum/<id>`, `/topic/<id>`, `/post/<id>`, `/<forum_id>/topic/new`, `/topic/<id>/post/new`, `/<forum_id>/markread`, `/search`, `/memberlist` e as de moderação (`/topic/<id>/lock`, `/hide`, `/delete`, `/highlight` etc.).
- Outros módulos que chamam os modelos direto: o `management` (painel de admin cria e edita fóruns e categorias), o `user` (`user/models.py`), o `utils/populate.py` (usado pelo comando `flaskbb populate`, que cria dados de exemplo) e os filtros do Jinja registrados no `app.py` (`topic_is_unread` e `forum_is_unread`).
- Configurações: o módulo lê várias opções do `flaskbb_config`, como `TRACKER_LENGTH`, `POSTS_PER_PAGE` e `TOPICS_PER_PAGE`.

### Saídas

- Banco de dados: grava nas tabelas `categories`, `forums`, `topics`, `posts`, `reports`, `topicsread`, `forumsread`, `moderators`, `topictracker` e `forumgroups`.
- Eventos pra plugins: o `Post.save` e o `Topic.save` disparam os hooks `flaskbb_event_post_save_before/after` e `flaskbb_event_topic_save_before/after` pelo pluggy.
- Telas: as views devolvem os templates `forum/*.html` e mensagens com `flash`.
- E-mail: o módulo forum não manda e-mail direto.

## Docstrings novas

Escrevi 3 docstrings em métodos do `flaskbb/forum/models.py` onde só o nome não dá pra entender o que o método faz. Estão no commit `18ce3f8` do meu fork.

| Método | Linha | O que não era óbvio |
|---|---|---|
| `Post._deal_with_last_post` | `models.py:397` | Tem que ser chamado antes de apagar ou esconder o post e não faz commit. Ele corrige o último post do tópico e, se for o caso, o do fórum. |
| `Topic.get_topic` | `models.py:678` | Dá 404 se o tópico não existe. Com `hiddencheck=True` o tópico escondido também dá 404 pra quem não pode ver conteúdo escondido. Com `False` (padrão) o tópico escondido volta normal. |
| `Topic.get_posts` | `models.py:698` | Não devolve só posts: os itens da paginação são pares `(Post, User)`, e o `User` pode ser `None` se o autor foi apagado. Os posts escondidos somem dependendo da permissão do usuário atual. |
