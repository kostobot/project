# Итоговое задание 5.1 (HW-03)

## Отчёт о реализации асинхронной работы с запросами в приложении NewsPortal.

### Задание 1. Реализовать рассылку уведомлений подписчикам категории после создания новости.

**Описание реализации:**
При добавлении поста в категорию срабатывает сигнал m2m_changed, который запускает Celery-задачу для асинхронной рассылки email-уведомлений всем подписчикам категории. Отправка писем выполняется в фоне через Redis + Celery.

**(`signals.py`):**
```python
@receiver(m2m_changed, sender=Post.category.through)
def notify_users_new_post(sender, instance, action, **kwargs):
    if action == 'post_add':
        send_new_post_notifications.delay(instance.id)
```


**(`tasks.py`):**
```python
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
```

**Результат:**
при публикации поста подписчики сразу получают email, задача выполняется асинхронно.


### Задание 2. Реализовать еженедельную рассылку с последними новостями (каждый понедельник в 8:00 утра)..

**Описание реализации:**
Через Celery Beat настроен планировщик, который каждую неделю запускает задачу send_weekly_digest. Она формирует подборку постов за последние 7 дней и отправляет письма подписчикам соответствующих категорий.

**(`celery.py`):**
```python
import os
from celery import Celery
from celery.schedules import crontab
 
os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'NewsPortal.settings')
 
app = Celery('NewsPortal')
app.config_from_object('django.conf:settings', namespace = 'CELERY')

app.autodiscover_tasks()

app.conf.beat_schedule = {
    'send-weekly-digest': {
        'task': 'blog.tasks.send_weekly_digest',
        'schedule': crontab(minute='00', hour='08', day_of_week='monday')
    },
}
```

**Результат:**
рассылка автоматически отправляется раз в неделю по расписанию.

**Запуск задач:**
python manage.py runserver
celery -A NewsPortal worker --pool=solo -l info
celery -A NewsPortal beat -l info
