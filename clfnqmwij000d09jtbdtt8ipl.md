---
title: "Scheduled Tasks in Django"
datePublished: Sat Mar 25 2023 08:57:21 GMT+0000 (Coordinated Universal Time)
cuid: clfnqmwij000d09jtbdtt8ipl
slug: scheduled-tasks-in-django
cover: https://cdn.hashnode.com/res/hashnode/image/stock/unsplash/BI465ksrlWs/upload/dfc4c8c372f20efe556452c7784ee5e3.jpeg
tags: celery, scheduler, django-background-tasks

---

When building applications, the need for certain background tasks comes into place.

Some of these tasks may be sending emails on a schedule, updating a model field or something else

In Django, we have several options to settle this need but our main concern for this post will be on apscheduler

This tutorial deals with showing you how to schedule tasks using APScheduler in Django

## Installation

```python
pip install apscheduler
```

When the installation is done, add apscheduler to requirements.txt

## Creating task

Here we are going to create the task to be executed in a **task.py** file

```python
def do_some_task():
  #background taskto do
  pass
```

When the task has been created, we move to the scheduling part

## Scheduling the task

It is not time to schedule the task in a new **scheduled.py** file

```python
from apscheduler.shedulers.background import BackgroundScheduler
from .task import do_some_task

def start_task():

    scheduler = BackgroundScheduler()
    scheduler.add_job(do_some_task, 'interval', minutes=30)
    scheduler.start()
```

we want this task to run on a specific interval so we included 'interval'. other choices to use include

`date:` use when you want to run the job just once at a certain point in time

`interval`: use when you want to run the job at fixed intervals of time

`cron`: use when you want to run the job periodically at a certain time(s) of the day

## Final steps

we have created the task and also set it to run at intervals, now we have to let django see the task, we call this in our **apps.py**

Note: **apps.py**, **scheduled.py** and **task.py** are in the same Django app

```python
class ProfilesConfig(AppConfig):
    default_auto_field = "django.db.models.BigAutoField"
    name = "profiles"

    def ready(self):
        from . import scheduled
        scheduled.start_task()
```

Thanks.

Read more on Apscheduler

[`https://apscheduler.readthedocs.io/en/3.x/userguide.html#adding-jobs`](https://apscheduler.readthedocs.io/en/3.x/userguide.html#adding-jobs)