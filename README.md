# Django Polls App

## Main Directory - Linking the URL for polls app
```python
from django.contrib import admin
from django.urls import path, include

urlpatterns = [
    path('polls/', include('polls.urls')),
    path('admin/', admin.site.urls),
]
```

## Including polls app in the list of installed apps in settings.py
```python
INSTALLED_APPS = [
    'polls.apps.PollsConfig', # polls app
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',
]
```
## apps.py in polls is configured like this
```python
from django.apps import AppConfig

class PollsConfig(AppConfig):
    name = 'polls'
```

## Question model in polls - models.py
```python
import datetime

from django.db import models
from django.utils import timezone


class Question(models.Model):
    def __str__(self):
        return self.question_text

    def was_published_recently(self):
        now = timezone.now()
        return now - datetime.timedelta(days = 1) <= self.pub_date <= now

    question_text = models.CharField(max_length = 200)
    pub_date = models.DateTimeField('date published')
    objects = models.Manager()
```

## admin.py in polls app - helps connect model to the database
Just the inclusion of one line `admin.site.register(Question)` connects it to the database.
Of course, we need to run `python manage.py makemigrations` to create empty table fields in the database and `python manage.py migrate` to make the database.
```python
from django.contrib import admin

from .models import Question

admin.site.register(Question)
```


## urls.py in polls app
```python
from django.urls import path
from . import views

app_name = 'polls'

urlpatterns = [
    # ex: /poll/
    path('', views.IndexView.as_view(), name="index"),

    # ex: /polls/5/
    path('<int:pk>', views.DetailView.as_view(), name="detail"),

    # ex: /polls/5/results/
    path('<int:pk>/results/', views.ResultsView.as_view(), name='results'),

    # ex: /polls/5/vote/
    path('<int:question_id>/vote/', views.vote, name='vote'),
]
```
## views.py in polls app: IndexView class to generate the generic view for index.html
```python
from django.http import HttpResponse, HttpResponseRedirect
from django.shortcuts import render, get_object_or_404
from django.urls import reverse
from django.views import generic
from django.utils import timezone

from .models import Question, Choice


class IndexView(generic.ListView):
    template_name = "polls/index.html"
    context_object_name = "latest_question_list"

    def get_queryset(self):
        """
        Return the last five published questions (not including those set to be published in the future). 
        """
        return Question.objects.filter(
            pub_date__lte=timezone.now()
        ).order_by('-pub_date')[:5]
```

## index.html for polls app look like this
```html
{% load static %}

<link rel="stylesheet" type="text/css" href="{% static 'polls/style.css' %}">
<script src="{% static 'polls/script.js' %}"></script>
<div>

  {% if latest_question_list %}
  <ul>
    {% for question in latest_question_list %}
    <li>
      <a href="{% url 'polls:detail' question.id %}">{{ question.question_text }}</a>
    </li>
    {% endfor %}
  </ul>
  {% else %}
  <p>No polls are available.</p>
  {% endif %}


</div>

<div id="root">

</div>
```

## QuestionModelTests(TestCase) in polls app
Tests:
- Check if a question with a date in the future is posted. Fails if it does.
- Check if a question older than a day is posted. Fails if it does.
- Check if a question less than a day old is posted. Passes if it does.
Similar logic has been used to test `IndexView` and `DetailView`.
```python
import datetime

from django.test import TestCase
from django.utils import timezone
from django.urls import reverse

from .models import Question

class QuestionModelTests(TestCase):

    
    def test_was_published_recently_with_future_question(self):
        """
        was_publishedP_recently() returns False for questions whose
        pub_date is in the future.
        """
        time = timezone.now() + datetime.timedelta(days = 30)
        future_question = Question(pub_date = time)
        self.assertIs(future_question.was_published_recently(), False)
    
    def test_was_published_recently_with_old_question(self):
        """
        was_published_recently() return False for question whose pub_date is older 1 day.
        """
        time = timezone.now() - datetime.timedelta(days = 1, seconds = 1)
        old_question = Question(pub_date = time)
        self.assertIs(old_question.was_published_recently(), False)

    def test_was_published_recently_with_recent_question(self):
        """
        was_published_recently() return True for questions whose pub_date is within the last day.
        """
        time = timezone.now() - datetime.timedelta(hours = 23, minutes = 59, seconds = 59)
        recent_question = Question(pub_date = time)
        self.assertIs(recent_question.was_published_recently(), True)
```