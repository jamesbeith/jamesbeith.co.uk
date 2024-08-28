---
title: "How to Enqueue Celery Tasks Using RabbitMQ Message Priorities"
date: "2024-08-28T13:30:00+10:00"
---

I have some slow running, low priority Celery tasks that I don’t want holding up more important tasks. I learnt how to configure Celery tasks, and queues, to support message priorities. Note, the project in question uses Django, Celery, and RabbitMQ.

Firstly, I needed to decide on my _priority values_. Whilst RabbitMQ supports priorities between 1 and 255, their docs highly recommend using values between 1 and 5.

<!-- vale off -->

> It is important to know that higher priority values require more CPU and memory resources, since RabbitMQ needs to internally maintain a sub-queue for each priority from 1, up to the maximum value configured for a given queue.  
> — https://www.rabbitmq.com/docs/priority

<!-- vale on -->

I decided on using values between 1 and 3, which conveniently map to low, medium and high priority.

```python
class Priority(enum.IntEnum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
```

I then needed to configure Celery by updating a few settings in my Django settings module. Note, I’m not using the `Priority` class here as that’s declared in another module that I didn’t want my Django settings module to depend upon.

```python
# The default priority for all tasks.
CELERY_TASK_DEFAULT_PRIORITY = 2
# The default max priority for all queues.
CELERY_TASK_QUEUE_MAX_PRIORITY = 3
```

I wanted to use the RabbitMQ priority feature on all my queues, but if I’d only wanted it on certain queues I could have configured those using `"x-max-priority"` like so.

```python
CELERY_QUEUE_DEFAULT = "celery"
CELERY_QUEUE_PRIORITY = "priority"
CELERY_TASK_QUEUES = [
    Queue(
        name=CELERY_QUEUE_DEFAULT,
        exchange=Exchange(CELERY_QUEUE_DEFAULT),
        routing_key=CELERY_QUEUE_DEFAULT,
    ),
    Queue(
        name=CELERY_QUEUE_PRIORITY,
        exchange=Exchange(CELERY_QUEUE_PRIORITY),
        routing_key=CELERY_QUEUE_PRIORITY,
        queue_arguments={"x-max-priority": 3}
    ),
]
```

After configuring Celery I ran into the following error starting up a worker.

```text
amqp.exceptions.PreconditionFailed: Queue.declare: (406) PRECONDITION_FAILED - inequivalent arg 'x-max-priority' for queue 'celery' in vhost '/': received the value '3' of type 'signedint' but current is none
```

This is because existing queues can’t change from a classic queue into priority queue, or vice versa. More specifically, the number of priorities a queue supports can’t change after queue declaration.

To resolve this issue, on my local machine I used the following `rabbitmqadmin` command to delete all my queues. Starting up a Celery worker created all my queues again now configured as priority queues.

```shell
# Caution! Delete all queues from RabbitMQ.
rabbitmqadmin --format=tsv --quiet list queues name | while read name; do rabbitmqadmin --quiet delete queue name=${name}; done
```

In production, the process was somewhat similar in that I created a new RabbitMQ instance and migrated the workers to point to the new instance. Again, starting up a production Celery worker created all my queues now configured as priority queues.

Lastly, I configured my appropriate Celery tasks using the `Priority` class.

```python
@app.task(priority=Priority.LOW)
def generate_report():
    ...


@app.task(priority=Priority.HIGH)
def send_email():
    ...
```

Some final notes. I’d already done so, but it’s worth disabling Celery [worker prefetching](https://docs.celeryq.dev/en/stable/userguide/configuration.html#worker-prefetch-multiplier) with priority queues so workers always fetch the highest prioritised task. Also, worth bearing in mind how priority queues [interact with other features](https://www.rabbitmq.com/docs/priority#interaction-with-other-features), in particular queues which have a max-length set.
