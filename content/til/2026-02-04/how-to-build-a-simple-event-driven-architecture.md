---
title: "How to build a simple event-driven architecture"
date: "2026-02-04T16:00:00+11:00"
---

I recently read this interesting [Reliable Django Signals](https://hakibenita.com/django-reliable-signals) article which left me inspired to have a play with building something similar. I wanted build a simple event-driven architecture where:

- Publishers can publish an event with a given payload
- One or many subscribers can subscribe to an event
- Events are asynchronous (as opposed to the synchronous [observer pattern](https://refactoring.guru/design-patterns/observer))

I took the opportunity to try out [`django-tasks`](https://github.com/RealOrangeOne/django-tasks) and [`blinker`](https://github.com/pallets-eco/blinker/) for this. Here’s what I came up with.

```python
import dataclasses
import functools
import pkgutil
from typing import Protocol

import blinker
from django_tasks import task


def publish(name: str, /, *, payload: dict[str, object]) -> None:
    """
    Publish an asynchronous event with the given `payload` for all receivers
    subscribed to `name`.
    """
    signal = _get_namespace().signal(name)
    for receiver in signal.receivers_for(blinker.ANY):
        receiver_identifier = _get_receiver_identifier(receiver)
        _execute_event.enqueue(
            receiver_identifier=receiver_identifier, name=name, payload=payload
        )


def subscribe(name: str, /, *, receiver: Receiver) -> None:
    """
    Subscribe a `receiver` callable for the given event `name`.
    """
    signal = _get_namespace().signal(name)
    signal.connect(receiver)


@dataclasses.dataclass(frozen=True, kw_only=True)
class Event:
    """
    The event object a receiver is called with.
    """

    name: str
    payload: dict[str, object]


class Receiver(Protocol):
    """
    Protocol for event receiver callables.
    """

    __qualname__: str

    def __call__(self, event: Event, /) -> None: ...


# Private


@functools.cache
def _get_namespace() -> blinker.Namespace:
    """
    Return the namespace used for all events.
    """
    return blinker.Namespace()


@task(queue_name="events")
def _execute_event(
    *, receiver_identifier: str, name: str, payload: dict[str, object]
) -> None:
    """
    Call the identified receiver with the event.
    """
    receiver_callable = _get_receiver_callable(receiver_identifier)
    receiver_callable(Event(name=name, payload=payload))


def _get_receiver_identifier(receiver: Receiver, /) -> str:
    """
    Return the identifier `<package>:<object>` for the given receiver.
    """
    return f"{receiver.__module__}:{receiver.__qualname__}"


def _get_receiver_callable(receiver_identifier: str, /) -> Receiver:
    """
    Return the receiver for the given identifier.
    """
    receiver_callable: Receiver = pkgutil.resolve_name(receiver_identifier)
    return receiver_callable
```

I also configured the follow Django settings for `django-tasks` . Specifically, setting up the dedicated `"events"` queue.

```diff
  INSTALLED_APPS = [
      ...,
+     "django_tasks",
+     "django_tasks.backends.database",
  }

+ TASKS = {
+     "default": {
+         "BACKEND": "django_tasks.backends.database.DatabaseBackend",
+         "QUEUES": ["events"],
+     }
+ }
```

To subscribe to events I could then use the following.

```python
from django import apps

import events


class Config(apps.AppConfig):
    ...

    def ready(self) -> None:
        events.subscribe("user.created", receiver=user_created)


def user_created(event: events.Event, /) -> None:
    ...
```

And the following to publish events.

```python
import events


def create_user(...) -> None:
    user = ...
    events.publish("user.created", payload={"user_id": user.id})
```

Thanks to [Haki](https://hakibenita.com/pages/about) for the original article, which is well worth the read.
