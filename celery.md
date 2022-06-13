# Django, Celery and RabbitMQ
A quick documentation for integrating celery & rabbitmq with a django project for asynchronous, background, adn periodic jobs.

**References:**
 - https://docs.celeryq.dev/en/latest/django/first-steps-with-django.html 
 - https://betterprogramming.pub/distributed-task-queues-with-celery-rabbitmq-django-703c7857fc17
 - https://simpleisbetterthancomplex.com/tutorial/2017/08/20/how-to-use-celery-with-django.html
 - [celery Beat] https://docs.celeryq.dev/en/latest/userguide/periodic-tasks.html
 - [celery Beat] https://django-celery-beat.readthedocs.io/en/latest/
 - [Daemonizing] https://docs.celeryq.dev/en/stable/userguide/daemonizing.html


## Integration Steps
Activate your virtual environment and change directory to your django project.

### Celery for asynchronous jobs
- Install python packages
  ```shell
  pip install celery
  pip install django-celery-results
  ```
  
- Configure settings in `settings.py` file:
  - Add `django_celery_results` to the `INSTALLED_APPS`.
  - Configure celery environment variables:
    ```python
    # ...
    # Celery
    CELERY_BROKER_URL = 'amqp://admin:admin@127.0.0.1:5672//'
    CELERY_CACHE_BACKEND = 'django-cache'
    CELERY_ACCEPT_CONTENT = ['application/json']
    CELERY_TASK_SERIALIZER = 'json'
    CELERY_TIMEZONE = "Asia/Kolkata"
  
    CELERY_RESULT_BACKEND = 'django-db'
    CELERY_RESULT_SERIALIZER = 'json'
    # DJANGO_CELERY_RESULTS_TASK_ID_MAX_LENGTH=191    # For Mysql
    # ...
    ```
    
- Create `celery.py` file inside the project app `<projectname>/celery.py`. 
  Refer the file [gagan_tutorial/celery.py](gagan_tutorial/celery.py) for the contents.
  
- Import the celery app in `<projectname>/__init__.py`. Refer [gagan_tutorial/\_\_init\_\_.py](gagan_tutorial/__init__.py)

- Create `task.py` inside any django app and define the task using the celery syntax.
  For instance, refer to the task file [gagan_tutorial/tasks.py](gagan_tutorial/tasks.py).
  
  **NOTE: Please make sure that your django app is included in `INSTALLED_APPS` otherwise the worker will not recognize the tasks.**
  
- Start the rabbit-mq server. For instance, consider the following docker command:

  https://hub.docker.com/_/rabbitmq
  ```shell
  docker run --hostname rabbitmq-dev --name rabbitmq_dev -e RABBITMQ_DEFAULT_USER=admin -e RABBITMQ_DEFAULT_PASS=admin -p 5672:5672 -p 15672:15672 -d rabbitmq:3-management
  ```
  
  Server: `amqp://admin:admin@127.0.0.1:5672//`
  
  Management Console: http://127.0.0.1:15672/
  
- Start the celery worker:
  ```shell
  celery -A gagan_tutorial worker --concurrency=4 -l INFO
  ```  
  **Note:** You may run more than one instance of the worker for better parallel processing.


- Send task to the queue:
  ```python
  from gagan_tutorial.tasks import *
  result = task_test.delay("hello")
  ```
- Open django admin and check the task results at http://127.0.0.1:8000/admin/django_celery_results/taskresult/.

- Open Rabbitmq management console http://127.0.0.1:15672/ to view server status.


## Celery Beat for periodic jobs
The following section discusses about enabling celert beat for scheduling periodic tasks using celery and rabbit-mq.
- Install python package
  ```shell 
  pip install django-celery-beat
  ```
  
- Configure `settings.py`
  - Add `django_celery_beat` in settings.py `INSTALLED_APPS`.
  - Configure celery beat scheduler
    ```python
    # ...
    CELERYBEAT_SCHEDULER = 'django_celery_beat.schedulers:DatabaseScheduler'
    # ...
    ```
    
- Migrate database
  ```shell
  python manage.py migrate
  ```
  
- Add task to the scheduler:
  
  Open django admin and create an entry for the task you want to put in the scheduler.

  _**Note:** The scheduler is expected to get updated as soon as the periodic task entry is changed from django admin._


- Run celery beat:
  Open a separate terminal (in addition to celery beat) and run the following command:
  ```shell
  celery -A gagan_tutorial beat -l INFO --scheduler django
  ```
  _OR_
  
  Run celery worker and beat together in one command (**development purpose only**)
  ```shell
  celery -A gagan_tutorial worker --beat --scheduler django --concurrency=4 -l INFO
  ```
  
  > :warning: Do not run more than one instance of celery beat process.
  
  
  ### Author
  Gagandeep Singh
  
  (singh.gagan144@gmail.com)
  
