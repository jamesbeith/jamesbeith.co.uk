---
title: "How to create a Django model index with an upper case empty string condition"
date: "2024-05-02T15:50:00+10:00"
aliases:
  - /til/2024-05-02/django-model-index-upper-case-empty-string-condition/
---

Let’s say I have the following Django model.

```python
from django.db import models

class User(models.Model):
    name = models.CharField(blank=True)
```

And I want to filter those like so.

```python
users = User.objects.filter(name__istartswith="John")
```

That would perform the following query.

```postgresql
SELECT "data_user"."id", "data_user"."name"
FROM "data_user"
WHERE UPPER("data_user"."name"::text) LIKE UPPER('John%')
```

I can improve the performance of that query by adding the following index to the model. To complement the use of `UPPER()` in the query I can use the `Upper()` function for the index’s expression.

```python
from django.db import models
from django.db.models.functions import Upper

class User(models.Model):
    name = models.CharField(blank=True)

    class Meta:
        indexes = [
            models.Index(Upper("name"), name="name_idx"),
        ]
```

As `name` could be blank, I can improve the index by adding the following condition to only index rows where the name is not blank.

```python
from django.db import models
from django.db.models.functions import Upper

class User(models.Model):
    name = models.CharField(blank=True)

    class Meta:
        indexes = [
            models.Index(
                Upper("name"),
                name="name_idx",
                condition=~models.Q(name=Upper(models.Value(""))),
            ),
        ]
```

Thanks to [Tim Bell](https://github.com/timb07) I learnt that for the condition I needed to pass `models.Value("")` as the argument to the `Upper()` function. Previously, I was only passing an empty string `condition=~models.Q(name=Upper(""))`, and whilst Django successfully created a migration for that index, when attempting to migrate I got the [following error raised](https://github.com/django/django/blob/5.0.4/django/db/models/sql/query.py#L1772-L1775).

```text
Operations to perform:
  Apply all migrations: admin, auth, contenttypes, data, sessions
Running migrations:
  Applying data.0001_initial...Traceback (most recent call last):
  File "project/src/manage.py", line 22, in <module>
    main()
  ...
  File "project/.venv/lib/python3.12/site-packages/django/db/models/sql/query.py", line 1772, in names_to_path
    raise FieldError(
django.core.exceptions.FieldError: Cannot resolve keyword '' into field. Choices are: id, question_text
```

It appears in this incorrect case `Upper("")` is referring to applying the function `Upper()` to the nonexistent model field named `""`.
