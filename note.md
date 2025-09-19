## ✅ 1. Start a Django Project

**If you don’t already have a Django project:**

```bash
python -m venv venv
source venv/bin/activate   # On Windows use: venv\Scripts\activate
pip install django
django-admin startproject myproject
cd myproject
```

## ✅ 2. Install Required Packages
```bash
pip install djangorestframework django-cors-headers celery drf-yasg
```


For RabbitMQ, you don’t install it with pip. It runs as a service — install instructions below.

## ✅ 3. Install and Start RabbitMQ
🐇 Install RabbitMQ (Linux/macOS/Windows)

**On Ubuntu/Debian:**
```bash
sudo apt-get install rabbitmq-server
sudo systemctl enable rabbitmq-server
sudo systemctl start rabbitmq-server
```

**On macOS (with Homebrew):**

```bash
brew install rabbitmq
brew services start rabbitmq
```

**On Windows:**

Download from: https://www.rabbitmq.com/download.html

Follow the installer instructions.

Once running, it listens at: amqp://localhost:5672

## ✅ 4. Configure Django Settings**
settings.py
**a. Add to INSTALLED_APPS:**
```python
INSTALLED_APPS = [
    ...
    'rest_framework',
    'corsheaders',
    'drf_yasg',
]
```

**b. Add Middleware for CORS:**

Put CorsMiddleware at the top, before CommonMiddleware.

```python
MIDDLEWARE = [
    'corsheaders.middleware.CorsMiddleware',
    'django.middleware.common.CommonMiddleware',
    ...
]
```

**c. CORS Settings:**

Allow all origins for development:
```python
CORS_ALLOW_ALL_ORIGINS = True
```
**d. Celery Settings:**

Add this to the bottom of settings.py:

```python
import os

CELERY_BROKER_URL = 'amqp://localhost'  # RabbitMQ default
CELERY_ACCEPT_CONTENT = ['json']
CELERY_TASK_SERIALIZER = 'json'
```

## ✅ 5. Create celery.py in your project folder

In your main Django project folder (myproject/myproject/), create celery.py:
```python
# myproject/celery.py
import os
from celery import Celery

os.environ.setdefault('DJANGO_SETTINGS_MODULE', 'myproject.settings')

app = Celery('myproject')
app.config_from_object('django.conf:settings', namespace='CELERY')
app.autodiscover_tasks()
```

Then in __init__.py of the same folder, import Celery:
```python
# myproject/__init__.py
from .celery import app as celery_app

__all__ = ['celery_app']
```
## ✅ 6. Run Celery Worker

In one terminal (with venv activated), run:
```bash
celery -A myproject worker --loglevel=info
```
## ✅ 7. Create a Test Task

In any Django app you’ve created:
```python
# myapp/tasks.py
from celery import shared_task

@shared_task
def add(x, y):
    return x + y
```

Then test in Django shell:
```bash
python manage.py shell

from myapp.tasks import add
add.delay(3, 4)  # Should return AsyncResult
```

## ✅ 8. Set Up drf-yasg for API Docs
**In urls.py of the root project (myproject/urls.py):**
```python
from django.contrib import admin
from django.urls import path, include
from rest_framework import permissions
from drf_yasg.views import get_schema_view
from drf_yasg import openapi

schema_view = get_schema_view(
   openapi.Info(
      title="My API",
      default_version='v1',
      description="API documentation with Swagger and ReDoc",
   ),
   public=True,
   permission_classes=[permissions.AllowAny],
)

urlpatterns = [
    path('admin/', admin.site.urls),
    path('api/', include('myapp.urls')),  # assuming you have an app
    path('swagger/', schema_view.with_ui('swagger', cache_timeout=0), name='schema-swagger-ui'),
    path('redoc/', schema_view.with_ui('redoc', cache_timeout=0), name='schema-redoc'),
]

```

Now visit:

http://localhost:8000/swagger/

http://localhost:8000/redoc/

## 🧪 Optional: Save Requirements
```bash
pip freeze > requirements.txt
```

## 🔧 INSTALLED TOOLS AND THEIR USES
### 1. Django REST Framework (djangorestframework)

**Purpose: Build RESTful APIs with Django.**

**Why You Need It:** It provides serializers, viewsets, routers, and other utilities to expose your models/data as APIs.

**Use Cases:**

Creating API endpoints for frontend/mobile apps.

Handling API authentication, pagination, filtering, and permissions.

Serializing Django models to JSON.

### 2. django-cors-headers

**Purpose: Handle CORS (Cross-Origin Resource Sharing).**

**Why You Need It:** If your frontend (e.g., React, Vue, Angular) runs on a different domain/port than your backend, you’ll get CORS errors without this.

**Use Cases:**

Allow cross-origin requests from frontend apps.

Prevent browser-side security errors when calling your API.

### 3. Celery

**Purpose: Run asynchronous background tasks outside of the request/response cycle.**

**Why You Need It:** Some tasks (like sending emails, processing files, or running long calculations) shouldn't block HTTP responses.

**Use Cases:**

Sending email notifications.

Processing uploaded images or files.

Scheduled tasks like nightly data cleanup.

Long-running ML model executions, video transcoding, etc.

### 4. RabbitMQ

**Purpose: Acts as a message broker for Celery.**

**Why You Need It:** Celery requires a broker to send and receive task messages. RabbitMQ handles this queueing efficiently.

**Use Cases:**

Reliable task queuing for Celery.

Delivering messages from Django (producer) to Celery workers (consumers).

Scaling out background workers on different machines or containers.

### 5. drf-yasg

**Purpose: Automatically generate Swagger/OpenAPI documentation for your APIs.**

**Why You Need It:** It saves time by automatically documenting your API endpoints based on DRF views and serializers.

**Use Cases:**

Generate interactive Swagger UI and Redoc documentation.

Share live API docs with frontend developers or third parties.

Test API endpoints directly from the browser.


## `seed.py` management command

Create file `listings/management/commands/seed.py`:

```python
# listings/management/commands/seed.py

import random
from datetime import timedelta
from django.core.management.base import BaseCommand
from django.utils import timezone
from faker import Faker

from django.contrib.auth import get_user_model
from listings.models import Listing

User = get_user_model()

class Command(BaseCommand):
    help = 'Seed the database with sample Listing data'

    def add_arguments(self, parser):
        parser.add_argument(
            '--number',
            type=int,
            default=10,
            help='Number of listings to create'
        )
        parser.add_argument(
            '--delete',
            action='store_true',
            help='Delete existing listings before seeding'
        )

    def handle(self, *args, **options):
        fake = Faker()
        number = options['number']
        do_delete = options['delete']

        if do_delete:
            self.stdout.write(self.style.WARNING('Deleting all existing Listing objects...'))
            Listing.objects.all().delete()

        users = list(User.objects.all())
        if not users:
            self.stdout.write(self.style.ERROR('No users found. Please create some users before running seed.'))
            return

        self.stdout.write(self.style.NOTICE(f'Seeding {number} listings...'))

        listings_to_create = []
        for _ in range(number):
            owner = random.choice(users)
            title = fake.sentence(nb_words=6)
            description = fake.paragraph(nb_sentences=3)
            price = round(random.uniform(10.0, 1000.0), 2)

            # For dates: start_date is some date in past or near future
            # ensure end_date > start_date
            start_date = fake.date_between(start_date='-30d', end_date='today')
            # ensure at least one day later
            end_date = fake.date_between(start_date=start_date + timedelta(days=1), end_date=start_date + timedelta(days=30))

            address = fake.address()

            listing = Listing(
                owner=owner,
                title=title,
                description=description,
                price=price,
                start_date=start_date,
                end_date=end_date,
                address=address,
                # created_at and updated_at handled automatically
            )
            listings_to_create.append(listing)

        Listing.objects.bulk_create(listings_to_create)

        self.stdout.write(self.style.SUCCESS(f'Successfully seeded {len(listings_to_create)} Listings.'))
```

### How to use

- Make sure you have the Faker library installed (e.g. `pip install Faker`).

- Make sure you have some User instances already, since Listing has an owner foreign key in this example.

- Run the command:

```bash
python manage.py seed --number 20
```

If you want to delete existing listings first:

```bash
python manage.py seed --number 20 --delete
```