---
title: "How to create a Django GeneratedField for extracting a timestamp from a UUIDv7"
date: "2025-12-04T16:30:00+11:00"
---

I recently read [How to use UUIDv7 in Python, Django and PostgreSQL](https://www.paulox.net/2025/11/14/how-to-use-uuidv7-in-python-django-and-postgresql/) from [Paolo Melchiorre](https://github.com/pauloxnet). I particularly liked the section demonstrating how to extend the model with a generated column that stores the timestamp extracted from the UUID.

```python
from django.db import models


class UUIDv7(models.Func):
    function = "uuidv7"
    output_field = models.UUIDField()


class UUIDExtractTimestamp(models.Func):
    function = "uuid_extract_timestamp"
    output_field = models.DateTimeField()


class Record(models.Model):
    uuid = models.UUIDField(db_default=UUIDv7(), primary_key=True)
    creation_time = models.GeneratedField(
        expression=UUIDExtractTimestamp("uuid"),
        output_field=models.DateTimeField(),
        db_persist=True,
    )
```

As Paolo says:

<!-- vale off -->

> A generated datetime column can be very useful even though UUIDv7 already embeds a timestamp. PostgreSQL computes it at write time, Django manages it declaratively through the ORM and having a proper datetime field makes filtering, ordering, indexing and using the Django admin much simpler without requiring annotations or extra computation on the UUID value.

<!-- vale on -->

[Read that specific section here](https://www.paulox.net/2025/11/14/how-to-use-uuidv7-in-python-django-and-postgresql/#adding-a-generatedfield-for-extracting-creation-time).
