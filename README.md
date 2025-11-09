# 📰 NewsPortal --- Асинхронные задачи с Celery и Redis

## 📌 Итоговое задание 5.1 (HW‑03)

------------------------------------------------------------------------

## 🎯 Цель

Реализовать асинхронную обработку задач в проекте **NewsPortal
(Django)** с использованием:

-   **Celery** --- для фоновой обработки задач
-   **Redis** --- как брокер задач
-   **Celery Beat** --- для планирования периодических задач

------------------------------------------------------------------------

## ✨ Реализованный функционал

1.  **Email‑уведомления подписчикам при появлении новой новости** (с
    использованием `m2m_changed` сигнала + Celery task).
2.  **Еженедельная рассылка дайджеста** новых постов в категориях, на
    которые подписан пользователь (каждый **понедельник в 8:00 MSK**).
3.  Асинхронная отправка писем **не блокирует основной поток
    приложения**.
4.  Полная интеграция **Django + Celery + Redis + Celery Beat**.

------------------------------------------------------------------------

## ⚙️ Настройка окружения

### 1. Установка зависимостей

``` sh
pip install celery redis django
```

### 2. Установка и запуск Redis

Если Redis не установлен:

``` sh
sudo apt update
sudo apt install redis
```

Запуск:

``` sh
sudo service redis-server start
```

------------------------------------------------------------------------

## 🛠 Конфигурация Django

### `settings.py`

``` python
CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'

CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Europe/Moscow'
```

### `celery.py` (в корне проекта)

``` python
import os
from celery import Celery
from celery.schedules import crontab

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'NewsPortal.settings')

app = Celery('NewsPortal')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()

app.conf.beat_schedule = {
    'send-weekly-digest': {
        'task': 'news.tasks.send_weekly_digest',
        'schedule': crontab(minute=0, hour=8, day_of_week='monday'),
    },
}
```

### `__init__.py`

``` python
from .celery import app as celery_app
__all__ = ('celery_app',)
```

------------------------------------------------------------------------

## 🔔 1. Уведомление при создании новости

### `signals.py`

``` python
@receiver(m2m_changed, sender=Post.category.through)
def notify_users_new_post(sender, instance, action, **kwargs):
    if action == 'post_add':
        send_new_post_notifications.delay(instance.id)
```

### `tasks.py`

``` python
@shared_task
def send_new_post_notifications(post_id):
    post = Post.objects.get(pk=post_id)
    categories = post.category.all()

    for category in categories:
        subscribers = category.subscribers.all()

        for user in subscribers:
            if not user.email:
                continue

            subject = f'Новый пост в категории: {category.name}'
            preview_text = post.text[:50] + ('...' if len(post.text) > 50 else '')

            text_content = (
                f'Здравствуй, {user.username}!\n'
                f'Новая статья в твоём любимом разделе \"{category.name}\": {post.title}\n\n'
                f'{preview_text}'
            )

            html_content = render_to_string(
                'subscribe_new_post.html',
                {'post': post, 'username': user.username, 'category': category.name}
            )

            email = EmailMultiAlternatives(
                subject=subject,
                body=text_content,
                from_email='your_email@example.com',
                to=[user.email],
            )
            email.attach_alternative(html_content, "text/html")
            email.send()
```

------------------------------------------------------------------------

## 📬 2. Еженедельный дайджест

### `tasks.py`

``` python
@shared_task
def send_weekly_digest():
    today = timezone.now()
    last_week = today - timedelta(days=7)
    posts = Post.objects.filter(created_at__gte=last_week)

    for category in Category.objects.all():
        category_posts = posts.filter(category=category)
        if not category_posts.exists():
            continue

        subscribers = category.subscribers.all()
        for user in subscribers:
            html_content = render_to_string(
                'weekly_digest.html',
                {'posts': category_posts, 'username': user.username, 'category': category.name}
            )

            email = EmailMultiAlternatives(
                subject=f'Подборка новостей за неделю — {category.name}',
                body='',
                from_email='your_email@example.com',
                to=[user.email],
            )
            email.attach_alternative(html_content, "text/html")
            email.send()
```

------------------------------------------------------------------------

## 🚀 Запуск проекта

Запустить **каждую команду в отдельном терминале**:

``` sh
# 1. Django сервер
python manage.py runserver

# 2. Celery worker
celery -A NewsPortal worker --pool=solo -l info

# 3. Планировщик (Celery Beat)
celery -A NewsPortal beat -l info
```

------------------------------------------------------------------------

## ✅ Результат

  Функция                               Статус
  ------------------------------------ --------
  Отправка email при создании поста       ✅
  Рассылка подписчикам по категориям      ✅
  Асинхронная обработка                   ✅
  Еженедельный дайджест (пн, 08:00)       ✅
  Redis как брокер                        ✅
  Celery Beat для планирования            ✅

------------------------------------------------------------------------

## ⭐ Готово!

Проект полностью настроен и выполняет задачи асинхронно, не блокируя
основной поток Django 🚀
