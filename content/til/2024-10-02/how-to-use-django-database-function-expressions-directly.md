---
title: "How to use Django database function expressions directly"
date: "2024-10-02T16:00:00+10:00"
---

Django has a many built-in [database functions](https://docs.djangoproject.com/en/5.1/ref/models/database-functions/) and a [documented](https://docs.djangoproject.com/en/5.1/ref/models/expressions/#django.db.models.Func) `Func` API for writing your own.

Whilst writing a custom `Func` subclass may sometimes be necessary, I learnt that there’s many cases when you can instantiate `Func` with the necessary arguments to get what you need. For example, take the following model.

```python
from django.contrib.postgres.fields import ArrayField
from django.db import models

class Post(models.Model):
    tags = ArrayField(models.CharField(max_length=200))
```

Making use of the `array_shuffle` Postgres [Array Function](https://www.postgresql.org/docs/current/functions-array.html) and an annotation, each of the post’s tags are in a randomised order.

```python
from django.db.models import F, Func

posts = Post.objects.annotate(
    tags_shuffle=Func(F("tags"), function="ARRAY_SHUFFLE"),
)
```

A second, slightly more contrived example.

```python
from django.db import models

class Post(models.Model):
    published_year = models.IntegerField()
    published_month = models.IntegerField()
    published_day = models.IntegerField()
```

This time using the `make_date` Postgres’ [Date/Time Function](https://www.postgresql.org/docs/current/functions-datetime.html) to get a `date` object from the post’s published values.

```python
from django.db.models import DateField, F, Func

posts = Post.objects.annotate(
    published_date=Func(
        F("published_year"),
        F("published_month"),
        F("published_day"),
        output_field=DateField(),
        function="MAKE_DATE",
    )
)
```
