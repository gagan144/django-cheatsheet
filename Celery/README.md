# Django, Celery and RabbitMQ
A quick documentation for integrating celery & rabbitmq with a django project for asynchronous, background, adn periodic jobs.

**Last Updated On**: 26-Jul-2022

**References:**
 - https://docs.celeryq.dev/en/latest/django/first-steps-with-django.html 
 - https://betterprogramming.pub/distributed-task-queues-with-celery-rabbitmq-django-703c7857fc17
 - https://simpleisbetterthancomplex.com/tutorial/2017/08/20/how-to-use-celery-with-django.html
 - [Celery Brokers] https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/index.html
 - [Celery Beat] https://docs.celeryq.dev/en/latest/userguide/periodic-tasks.html
 - [Celery Beat] https://django-celery-beat.readthedocs.io/en/latest/
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
    CELERY_RESULT_EXTENDED = True  # To allow django_celery_results to store task in progress
    CELERY_TASK_TRACK_STARTED = True  # To allow django_celery_results to store task in progress
    # DJANGO_CELERY_RESULTS_TASK_ID_MAX_LENGTH=191    # For Mysql
    # ...
    ```
    
    - To use **AmazonSQS** as the message broker, 
      
      Install celery with sqs:
      ```shell
      pip install celery[sqs]
      ```
      and use the following settings:
      ```python
      from kombu.utils.url import safequote

      # ...
      CELERY_BROKER_URL = "sqs://{access_key_id}:{secret_access_key}@".format(
          access_key_id=safequote("---access_key_id---"),
          secret_access_key=safequote("---secret_access_key---")
      )
      CELERY_BROKER_TRANSPORT_OPTIONS = {
          "region": "---region---",
          "visibility_timeout": 3600,  # 1 Hour
          "polling_interval": 0.3,    # In seconds
          "queue_name_prefix": "<projectname>-dev-"
      }
      CELERY_TIMEZONE = TIME_ZONE
      # ...
      ```
    
- Create `celery.py` file inside the project app `<projectname>/celery.py` with the following content: 
  ```python
  import os
  from celery import Celery

  # Set the default Django settings module for the 'celery' program.
  os.environ.setdefault('DJANGO_SETTINGS_MODULE', '<project>.settings')

  # Create an instance celery app
  # app = Celery('<project>')
  app = Celery('<project>', result_extended=True) # To allow django_celery_results to store task in progress

  # Using a string here means the worker doesn't have to serialize
  # the configuration object to child processes.
  # - namespace='CELERY' means all celery-related configuration keys
  #   should have a `CELERY_` prefix.
  app.config_from_object('django.conf:settings', namespace='CELERY')

  # Load task modules from all registered Django apps.
  app.autodiscover_tasks()

  # Create a debug task for testing
  @app.task(bind=True)
  def debug_task(self):
      print(f'Debug Task - Request: {self.request!r}')
      
  ```
  
- Import the celery app in `<projectname>/__init__.py` with the following content:
  ```python
  # This will make sure the app is always imported when
  # Django starts so that shared_task will use this app.
  from .celery import app as celery_app

  __all__ = ('celery_app',)
  ```

- Create `tasks.py` inside any django app and define the task using the celery syntax. For instance:
  ```python
  import os
  from django.contrib.auth.models import User
  from django.utils import timezone
  from django.conf import settings
  from celery import shared_task


  @shared_task(bind=True, track_started=True)
  def task_test(self, message=None):
      """
      An asynchronous example task that counts the number of active users in the system.
      """
      count_all_users = User.objects.filter(is_active=True).count()
      count_staff_users = User.objects.filter(is_active=True, is_staff=True).count()
      count_super_users = User.objects.filter(is_active=True, is_superuser=True).count()

      return {
          "message_echo": message,
          "count_all_users": count_all_users,
          "count_staff_users": count_staff_users,
          "count_super_users": count_super_users
      }


  @shared_task
  def task_test_generate_report(remarks=None):
      """
      A periodic example task to generate user report.
      """
      now = timezone.now()

      # Query Database
      count_all_users = User.objects.filter(is_active=True).count()
      count_staff_users = User.objects.filter(is_active=True, is_staff=True).count()
      count_super_users = User.objects.filter(is_active=True, is_superuser=True).count()

      # Create CSV content
      content = "\n".join([
          "Datetime-UTC,Count-All-Users,Count-Staff-Users,Count-Super-Users,Remarks",
          f"{now.strftime('%Y-%m-%dT%H:%M:%SZ')},{count_all_users},{count_staff_users},{count_super_users},{remarks if remarks else ''}"
      ])

      # Save report
      dir_report = os.path.join(settings.BASE_DIR, "media", "user-reports")
      path_report = os.path.join(dir_report, f"user-report-{now.strftime('%Y_%m_%d_%H_%M_%S')}.csv")
      if not os.path.exists(dir_report):
          os.makedirs(dir_report)

      with open(path_report, "w") as f:
          f.write(content)

      # return
      return {
          "path_report": path_report
      }

  ```
  
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
  celery -A <projectname> worker --concurrency=4 -l INFO
  ```  
  **Note:** You may run more than one instance of the worker for better parallel processing.


- Send task to the queue:
  ```python
  from myapp.tasks import *
  result = task_test.delay("hello")
  ```
- Open django admin and check the task results at http://127.0.0.1:8000/admin/django_celery_results/taskresult/.

- Open Rabbitmq management console http://127.0.0.1:15672/ to view server status.


### Celery Beat for periodic jobs
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
  celery -A <projectname> beat -l INFO --scheduler django
  ```
  _OR_
  
  Run celery worker and beat together in one command (**development purpose only**)
  ```shell
  celery -A <projectname> worker --beat --scheduler django --concurrency=4 -l INFO
  ```
  
  > :warning: Do not run more than one instance of celery beat process.


### Daemonizing Celery worker and beat on Linux servers through systemctl

- **Create directories for logs with proper permissions**
  ```shell
  sudo mkdir /var/log/celery /var/run/celery
  sudo chown <user>:<group> /var/log/celery /var/run/celery 
  ```

- **Celery configuration file**
  
  Create celery configuration file at `/etc/conf.d/celery` as follows and set 
  the values for `CELERY_BIN` and `DJANGO_SETTINGS_MODULE` as per yor project:

  ```dotenv
  # Name of nodes to start
  # here we have a single node
  CELERYD_NODES="w1"
  # or we could have three nodes:
  # CELERYD_NODES="w1 w2 w3"
  
  # Absolute or relative path to the 'celery' command:
  CELERY_BIN="/path/to/venv/bin/celery"
  
  # App instance to use
  # comment out this line if you don't use an app
  CELERY_APP="<projectname>"
  # or fully qualified:
  # CELERY_APP="proj.tasks:app"
  
  # How to call manage.py
  CELERYD_MULTI="multi"
  
  # Extra command-line arguments to the worker
  # Note: Concurrency sets the no of worker to spwan 
  CELERYD_OPTS="--time-limit=300 --concurrency=3"
  
  # Log and PID files
  # - %n will be replaced with the first part of the nodename.
  # - %I will be replaced with the current child process index
  #   and is important when using the prefork pool to avoid race conditions.
  CELERYD_PID_FILE="/var/run/celery/%n.pid"
  CELERYD_LOG_FILE="/var/log/celery/%n%I.log"
  CELERYD_LOG_LEVEL="INFO"
  
  # You may wish to add these options for Celery Beat
  CELERYBEAT_PID_FILE="/var/run/celery/beat.pid"
  CELERYBEAT_LOG_FILE="/var/log/celery/beat.log"
  
  # Django settings
  DJANGO_SETTINGS_MODULE="<projectname>.settings"
  ```
  
- **Celery Worker service**
  
  Create a Linux system service file at `/etc/systemd/system/celery.service` with
  the following content and set values as per your project:
  ```unit file (systemd)
  [Unit]
  Description=Celery Worker Service
  After=network.target
  
  [Service]
  Type=forking
  User=<user>
  Group=<group>
  EnvironmentFile=/etc/conf.d/celery
  WorkingDirectory=/path/to/django-project
  ExecStart=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi start $CELERYD_NODES \
      --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
      --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
  ExecStop=/bin/sh -c '${CELERY_BIN} multi stopwait $CELERYD_NODES \
      --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
      --loglevel="${CELERYD_LOG_LEVEL}"'
  ExecReload=/bin/sh -c '${CELERY_BIN} -A $CELERY_APP multi restart $CELERYD_NODES \
      --pidfile=${CELERYD_PID_FILE} --logfile=${CELERYD_LOG_FILE} \
      --loglevel="${CELERYD_LOG_LEVEL}" $CELERYD_OPTS'
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  ```

- **Celery Beat service**
  
  Create another Linux system service file at `/etc/systemd/system/celerybeat.service` with
  the following content and set values as per your project:
  ```unit file (systemd)
  [Unit]
  Description=Celery Beat Service
  After=network.target
  
  [Service]
  Type=simple
  User=<user>
  Group=<group>
  EnvironmentFile=/etc/conf.d/celery
  WorkingDirectory=/path/to/django-project
  ExecStart=/bin/sh -c '${CELERY_BIN} -A ${CELERY_APP} beat \
      --pidfile=${CELERYBEAT_PID_FILE} \
      --logfile=${CELERYBEAT_LOG_FILE} --loglevel=${CELERYD_LOG_LEVEL} \
      --scheduler django'
  Restart=always
  
  [Install]
  WantedBy=multi-user.target
  ```

- **Reload the systemctl daemon**:
  ```shell
  sudo systemctl daemon-reload
  ```
  
- **Enable the services so that it doesn't get disabled if the server restarts**:
  ```shell
  sudo systemctl enable celery.service
  sudo systemctl enable celerybeat.service
  ```

- **Start your service**:
  ```shell
  sudo systemctl start celery.service
  sudo systemctl start celerybeat.service
  ```
  
  Check status:
  ```shell
  sudo systemctl status celery.service
  sudo systemctl status celerybeat.service
  ```

#### Manage services
```shell
sudo systemctl start|stop|restart|status <servicename>.service
```
OR
```shell
sudo systemctl start|stop|restart|status <servicename>
```

  
## Key Points 

### Start celery worker/beat process:
  
- Development only (Run worker and beat together)
  ```shell
  celery -A <projectname> worker --beat --scheduler django --concurrency=4 -l INFO
  ```
  
- Production usage:
  - Worker:
    ```shell
    celery -A <projectname> worker --concurrency=4 -l INFO
    ```
    
  - Beat scheduler:
    ```shell
    celery -A <projectname> beat -l INFO --scheduler django
    ```

- Manage celery services
  ```shell
  sudo systemctl start|stop|restart|status <servicename>.service
  ```
  
## Author
Gagandeep Singh
(singh.gagan144@gmail.com)
