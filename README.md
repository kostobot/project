📰 NewsPortal
Итоговое задание 5.1 (HW-03)
📌 Тема: Асинхронная обработка задач в Django с использованием Celery и Redis
🎯 Цель работы

Реализовать асинхронную обработку задач в Django-проекте NewsPortal с использованием Celery и Redis.

Основные задачи:

Реализовать рассылку уведомлений подписчикам после создания новости.

Настроить еженедельную рассылку новых публикаций (каждый понедельник в 8:00 утра).

⚙️ Предварительная настройка системы

Перед реализацией функционала выполнены следующие шаги:

Установлен Redis.

Установлен Celery.

В settings.py добавлены параметры конфигурации Celery:

CELERY_BROKER_URL = 'redis://localhost:6379/0'
CELERY_RESULT_BACKEND = 'redis://localhost:6379/0'

CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
CELERY_RESULT_SERIALIZER = 'json'
CELERY_TIMEZONE = 'Europe/Moscow'


В корне проекта создан файл celery.py, а в __init__.py добавлена инициализация:

from .celery import app as celery_app

__all__ = ('celery_app',)

📨 Задание 1. Рассылка уведомлений подписчикам после создания новости
🧩 Описание реализации

Механизм реализован с использованием:

Сигнала Django m2m_changed — для отслеживания добавления новости в категорию.

Celery — для асинхронной отправки писем подписчикам.

Redis — как брокера задач.

📄 signals.py
from django.db.models.signals import m2m_changed
from django.dispatch import receiver
from .models import Post
from .tasks import send_new_post_notifications

@receiver(m2m_changed, sender=Post.category.through)
def notify_users_new_post(sender, instance, action, **kwargs):
    if action == 'post_add':
        send_new_post_notifications.delay(instance.id)

📄 tasks.py
from celery import shared_task
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string
from .models import Post

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
                f'Новая статья в твоём любимом разделе "{category.name}": {post.title}\n\n'
                f'{preview_text}'
            )

            html_content = render_to_string(
                'subscribe_new_post.html',
                {'post': post, 'username': user.username, 'category': category.name}
            )

            email = EmailMultiAlternatives(
                subject=subject,
                body=text_content,
                from_email='kastetpsy@yandex.ru',
                to=[user.email],
            )
            email.attach_alternative(html_content, "text/html")
            email.send()

✅ Результат

После добавления новой статьи подписчики соответствующей категории получают уведомление на email с кратким описанием новости и ссылкой на её полную версию.
Отправка писем выполняется асинхронно через Celery, что не блокирует основной поток приложения.

🗓 Задание 2. Еженедельная рассылка новостей
🧩 Описание реализации

Для еженедельной рассылки используется Celery Beat.
Задача выполняется каждый понедельник в 8:00 утра и формирует подборку новостей за последнюю неделю, отправляя их подписчикам соответствующих категорий.

📄 celery.py
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

📄 tasks.py
from datetime import timedelta
from django.utils import timezone
from django.core.mail import EmailMultiAlternatives
from django.template.loader import render_to_string
from .models import Post, Category

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
                from_email='kastetpsy@yandex.ru',
                to=[user.email],
            )
            email.attach_alternative(html_content, "text/html")
            email.send()

⚙️ Запуск процессов

Для корректной работы системы необходимо запустить три процесса в отдельных терминалах:

# 1. Запуск сервера Django
python manage.py runserver

# 2. Запуск Celery worker
celery -A NewsPortal worker --pool=solo -l info

# 3. Запуск Celery beat (планировщик)
celery -A NewsPortal beat -l info

📊 Результаты выполнения

При создании новой новости подписчики категории получают уведомление на email.

Каждую неделю (понедельник, 08:00) подписчики получают подборку свежих статей.

Отправка писем выполняется асинхронно, не влияя на производительность веб-приложения.

Настроена связка Django + Celery + Redis, реализована работа с задачами и планировщиком.

🧩 Вывод

В ходе выполнения проекта:

Реализована асинхронная архитектура с Celery и Redis;

Настроена автоматическая рассылка уведомлений и еженедельных подборок;

Повышена производительность и масштабируемость приложения.

👨‍💻 Автор: Твоё имя или никнейм
📅 Дата: Ноябрь 2025
🚀 Технологии: Django, Celery, Redis, SMTP, HTML Templates
