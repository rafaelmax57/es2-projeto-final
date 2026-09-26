# Catálogo de code smells - Parte 2

Arquivo escolhido: `flaskbb/forum/models.py`

Continuei no mesmo arquivo da Parte 1 porque já conhecia ele e os testes que escrevi ajudam a garantir que as refatorações não quebram nada. As linhas abaixo são do commit `71b6ee9` do meu fork (antes de qualquer refatoração).

---

## 1. Long Method - `Forum.update_read`

**Onde:** `flaskbb/forum/models.py:1177-1273`

```python
    def update_read(
        self, user: "User", forumsread: ForumsRead | None, topicsread: TopicsRead | None
    ):
        ...
        if not user.is_authenticated or topicsread is None:
            return False

        read_cutoff = None
        if flaskbb_config["TRACKER_LENGTH"] > 0:
            read_cutoff = time_utcnow() - timedelta(
                days=flaskbb_config["TRACKER_LENGTH"]
            )

        # fetch the unread posts in the forum
        unread_count = db.session.execute(
            db.select(db.func.count())
            .select_from(Topic)
            .outerjoin(
                TopicsRead,
                db.and_(TopicsRead.topic_id == Topic.id, TopicsRead.user_id == user.id),
            )
            ...
        ).scalar_one()

        # No unread topics available - trying to mark the forum as read
        if unread_count == 0:
            ...
```

**Por que é smell:** o método tem quase 100 linhas e faz várias coisas diferentes: valida o usuário, calcula a data de corte do tracker, monta uma consulta grande com dois `outerjoin` pra contar tópicos não lidos e ainda decide se cria ou atualiza o `ForumsRead`. Pra entender a regra de "quando o fórum fica como lido" tem que passar por cima da query inteira.

---

## 2. Duplicated Code - cálculo do `read_cutoff`

**Onde:** `flaskbb/forum/models.py:701-705` (`Topic.tracker_needs_update`) e `flaskbb/forum/models.py:1201-1205` (`Forum.update_read`)

```python
        read_cutoff = None
        if flaskbb_config["TRACKER_LENGTH"] > 0:
            read_cutoff = time_utcnow() - timedelta(
                days=flaskbb_config["TRACKER_LENGTH"]
            )
```

**Por que é smell:** o mesmo bloco aparece igualzinho em duas classes. Se a regra do tracker mudar (por exemplo, contar em horas em vez de dias), tem que lembrar de mudar nos dois lugares, e é fácil esquecer um.

---

## 3. Duplicated Code - zerar o último post do fórum

**Onde:** `flaskbb/forum/models.py:414-418` (`Post._deal_with_last_post`), `flaskbb/forum/models.py:959-963` (`Topic._remove_topic_from_forum`) e `flaskbb/forum/models.py:1168-1172` (`Forum.update_last_post`)

```python
            self.last_post = None
            self.last_post_title = None
            self.last_post_user = None
            self.last_post_username = None
            self.last_post_created = None
```

(nos outros dois lugares é igual, só que com `self.topic.forum.` ou `self.forum.` na frente)

**Por que é smell:** são as mesmas 5 linhas repetidas em três métodos de três classes diferentes. Se o fórum ganhar mais um campo de "último post", tem que alterar os três, e se esquecer um o fórum fica com dado velho.

---

## 4. Dead Code - `else` que nunca executa em `Topic.update_read`

**Onde:** `flaskbb/forum/models.py:758-789`

```python
        updated = False

        if topicsread:
            ...
            updated = True

        elif not topicsread:
            ...
            updated = True

        # No unread posts
        else:
            updated = False

        # Save True/False if the forums tracker has been updated.
        updated = forum.update_read(user, forumsread, topicsread)

        return updated
```

**Por que é smell:** `if topicsread` e `elif not topicsread` já cobrem todos os casos, então o `else` nunca roda. Na cobertura da Parte 1 a linha 784 (`updated = False` do `else`) sempre aparecia como não coberta, e agora sei o motivo. Além disso, o valor de `updated` que é calculado nos `if` é jogado fora, porque na linha 787 ele é sobrescrito pelo retorno do `forum.update_read`. O comentário "No unread posts" também engana, porque esse caso nunca acontece ali.

---

## 5. Data Clumps - campos de "último post" do fórum

**Onde:** `flaskbb/forum/models.py:325-329`, `408-412`, `501-505`, `953-957`, `1022-1026` e `1160-1164`

```python
                    topic.forum.last_post = self
                    topic.forum.last_post_user = self.user
                    topic.forum.last_post_title = topic.title
                    topic.forum.last_post_username = user.username
                    topic.forum.last_post_created = created
```

**Por que é smell:** `last_post`, `last_post_title`, `last_post_user`, `last_post_username` e `last_post_created` andam sempre juntos, são atribuídos juntos em seis lugares diferentes. Isso mostra que eles são um conceito só ("o último post do fórum") que não tem um lugar próprio no código. O smell 3 é um caso particular disso (quando os cinco viram `None`).

---

## 6. Feature Envy / Message Chains - `Post._deal_with_last_post`

**Onde:** `flaskbb/forum/models.py:388-430`

```python
    def _deal_with_last_post(self):
        if self.topic.last_post == self:
            # update the last post in the forum
            if self.topic.last_post == self.topic.forum.last_post:
                ...
                if second_last_post:
                    # now lets update the second last post to the last post
                    self.topic.forum.last_post = second_last_post
                    self.topic.forum.last_post_title = second_last_post.topic.title  # noqa
                    self.topic.forum.last_post_user = second_last_post.user
                    ...
```

**Por que é smell:** é um método do `Post`, mas quase não usa nada do próprio post. Ele mexe o tempo todo nos dados do `Topic` e do `Forum` usando cadeias como `self.topic.forum.last_post_title`. O método tem "inveja" do `Forum`, e essa lógica de trocar o último post deveria ser responsabilidade do próprio fórum. Tanto que a linha ficou tão grande que precisou de `# noqa` pra passar no linter.

---

## 7. Comentário desatualizado - docstring do `Forum.save`

**Onde:** `flaskbb/forum/models.py:1305-1312`

```python
    @override
    def save(self, groups: "Group | None" = None):
        """Saves a forum

        :param moderators: If given, it will update the moderators in this
                           forum with the given iterable of user objects.
        :param groups: A list with group objects.
        """
```

**Por que é smell:** a docstring explica um parâmetro `moderators` que não existe mais no método. Quem lê acha que dá pra passar moderadores pro `save`, mas não dá. Comentário errado é pior que nenhum comentário.

---

## 8. Nome enganoso (erro de digitação) - `invovled_users`

**Onde:** `flaskbb/forum/models.py:890` e `894` (`Topic.delete`)

```python
        # get the users before deleting the topic
        invovled_users = self.involved_users()

        topic_last_post_id = self.last_post_id
        db.session.delete(self)
        self._fix_user_post_counts(invovled_users)
```

**Por que é smell:** o nome da variável está escrito errado (`invovled` em vez de `involved`). No `Topic.hide` e no `Topic.unhide` a mesma variável se chama `involved_users`, então quem procura pelo nome certo não acha esse lugar. É pequeno, mas atrapalha a busca e a leitura.
