# 📰 NewsPortal

## Итоговое задание 5.1 (HW‑03)

### 📌 Тема

Асинхронная обработка задач в Django с использованием **Celery** и
**Redis**

------------------------------------------------------------------------

## 🎯 Цель

-   Отправка email подписчикам при публикации новости\
-   Еженедельный дайджест (каждый **понедельник в 8:00**)

------------------------------------------------------------------------

## ⚙️ Настройка

### 1) Установка

``` bash
pip install celery redis
```

### 2) Переменная окружения для Redis Cloud

``` bash
export REDIS_CLOUD="ВАШ_ПАРОЛЬ_ОТ_REDIS_CLOUD"
```

Windows (PowerShell):

``` powershell
setx REDIS_CLOUD "ВАШ_ПАРОЛЬ_ОТ_REDIS_CLOUD"
```

### 3) settings.py

``` python
import os

CELERY_BROKER_URL = f"redis://:{os.environ.get('REDIS_CLOUD')}@redis-10218.c14.us-east-1-2.ec2.redns.redis-cloud.com:10218"
CELERY_RESULT_BACKEND = f"redis://:{os.environ.get('REDIS_CLOUD')}@redis-10218.c14.us-east-1-2.ec2.redns.redis-cloud.com:10218"

CELERY_ACCEPT_CONTENT = ['application/json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Europe/Moscow'
```

### 4) celery.py (в корне проекта)

``` python
import os
from celery import Celery
from celery.schedules import crontab

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'NewsPortal.settings')

app = Celery('NewsPortal')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

app.conf.beat_schedule = {
    'weekly-digest': {
        'task': 'news.tasks.send_weekly_digest',
        'schedule': crontab(hour=8, minute=0, day_of_week='monday'),
    },
}
```

### 5) **init**.py

``` python
from .celery import app as celery_app
__all__ = ('celery_app',)
```

------------------------------------------------------------------------

## 📨 Уведомление о новой новости

**signals.py**

``` python
@receiver(m2m_changed, sender=Post.category.through)
def notify_users_new_post(sender, instance, action, **kwargs):
    if action == 'post_add':
        send_new_post_notifications.delay(instance.id)
```

**tasks.py**

``` python
@shared_task
def send_new_post_notifications(post_id):
    post = Post.objects.get(pk=post_id)
    for cat in post.category.all():
        for user in cat.subscribers.all():
            if user.email:
                send_mail(
                    f"Новый пост: {post.title}",
                    post.text[:100],
                    "from@mail.com",
                    [user.email]
                )
```

------------------------------------------------------------------------

## 🗓 Еженедельный дайджест

**tasks.py**

``` python
@shared_task
def send_weekly_digest():
    week = timezone.now() - timedelta(days=7)
    posts = Post.objects.filter(created_at__gte=week)

    for cat in Category.objects.all():
        cat_posts = posts.filter(category=cat)
        if cat_posts.exists():
            for user in cat.subscribers.all():
                if user.email:
                    send_mail(
                        f"Дайджест за неделю: {cat.name}",
                        "\n".join(p.title for p in cat_posts),
                        "from@mail.com",
                        [user.email]
                    )
```

------------------------------------------------------------------------

## ▶️ Запуск

``` bash
python manage.py runserver
celery -A NewsPortal worker --pool=solo -l info
celery -A NewsPortal beat -l info
```

------------------------------------------------------------------------

## ✅ Итог

-   Асинхронные письма ✅\
-   Планировщик рассылки ✅\
-   Интеграция Django + Celery + Redis ✅
