# cronjob within a docker container
Showing that is possible, if it is a good idea, that's up to you :)


## Scenario
A python application is running within a container 24/7. An update job is required
to fetch new data once a week (Sunday was chosen). After the update, the application must be notified to pick up the changes and switch to the new data. All data is stored in a database, so the application essentially needs to switch the table.

Different tables are created to track changes, and while overriding an existing table is an option, creating new tables is preferred. This approach ensures that, in case of any errors, the application can easily revert to the previous table without restoring a prior backup.

**NOTE**: Ultimately, setting up a cronjob within a Docker container alongside the application felt wrong, so a different approach was chosen.


## Simple DIY example

Dockerfile
```Dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get -y install cron

WORKDIR /usr/src/app

RUN echo "#!/bin/bash" > tester_job.sh
RUN echo "echo kw >> /root/test" >> tester_job.sh
RUN chmod +x tester_job.sh

RUN echo "* * * * * root /usr/src/app/tester_job.sh" > cronjobs_config
RUN cp cronjobs_config /etc/cron.d/
RUN chmod 0644 /etc/cron.d/cronjobs_config
RUN crontab /etc/cron.d/cronjobs_config

RUN touch /var/log/cron.log
CMD ["/bin/bash", "-c", "cron; tail -f /var/log/cron.log"]
```

**NOTE** `cron` is not started per default as a container does not have a full
init system. This can be verified by running `pidof cron` inside the container it will
not return a PID if not explicitly started.

* Build: `docker build -t cron_sample .`
* Run: `docker run -d cron_sample`
* ContainerID: `docker ps | grep cron_sample`
* Attach to container: `docker exec -it <container-id> /bin/bash`
* Stop container: `docker stop <container-id> -t 0`

A process is required to keep the container alive, hence the `tail` command is used.

Given the scenario, the application needs to remain alive. Let’s tweak the Dockerfile.

Sample application that keeps the docker alive.
```Python
import time
while True:
    time.sleep(1)
```

Replace/extend the Dockerfile with
```diff
+ COPY py_loop.py py_loop.py

- CMD ["/bin/bash", "-c", "cron; tail -f /var/log/cron.log"]
+ CMD ["/bin/bash", "-c", "cron; python3 /usr/src/app/py_loop.py"]
```

This approach achieves the same result. Next, replace the `echo` command with a Python script.

Python cronjob script `py_cron.py`:
```Python
#!/usr/bin/python3

with open("/root/test", "a+") as f:
    f.write("kw\n")
```

```diff
+ COPY py_cron.py py_cron.py
+ RUN chmod +x py_cron.py

- RUN echo "#!/bin/bash" > tester_job.sh
- RUN echo "echo hi >> /root/test" >> tester_job.sh
- RUN chmod +x tester_job.sh

- RUN echo "* * * * * root /usr/src/app/tester_job.sh" > cronjobs_config
+ RUN echo "* * * * * root /usr/src/app/py_cron.py" > cronjobs_config
```


## Be Careful
Cronjobs within a docker container do behave a little bit different than
cronjobs on a "regular" host.

1. **Location**: `crontab` config files needs be located within `/etc/cron.d/`. If not
it will fail silently. Even though `crontab -l` will show the jobs and `cron`
is running.

2. The root user is required, if not provided, same behaviour, a silent fail.
(This might be related as the config is moved to `/etc/cron.d` and is therefore
a systemwide cronjob, and those need to have a user specified)


## Summary
To keep the application and the updater separated, cause it is possible
the that updater code might change (e.g. data preprocessing updates) and the
application code not.
Also having multiple docker containers running, and only some having cronjobs
seems not ideal this would require good documentation to avoid confusion.

To have all production cronjobs available at one sight the decision was made
to assign them all to the production user. This gives one central location for
all jobs and not having them separated between containers.

Final Dockerfile
```Dockerfile
FROM ubuntu:latest

RUN apt-get update && apt-get -y install cron

WORKDIR /usr/src/app

COPY py_loop.py py_loop.py
COPY py_cron.py py_cron.py

RUN chmod +x py_cron.py

RUN echo "* * * * * root /usr/src/app/py_cron.py"  > cronjobs_config
RUN cp cronjobs_config /etc/cron.d/
RUN chmod 0644 /etc/cron.d/cronjobs_config

CMD ["/bin/bash", "-c", "cron; python3 /usr/src/app/py_loop.py"]
```

----
written 4.12.2024
