# paperless-ng-dockerfiles

> The following was lifted largely from [the Paperless-ng project](https://github.com/jonaswinkler/paperless-ng) and specifically from its [./docs/setup.rst](https://github.com/jonaswinkler/paperless-ng/docs/setup.rst) document.

## Download

Go to the project page on GitHub and download the
[latest release](https://github.com/jonaswinkler/paperless-ng/releases).
There are multiple options available but I specifically chose to pull the latest image from Docker Hub, so...

*   Download the dockerfiles archive if you want to pull paperless from
    Docker Hub.


## Overview of Paperless-ng

Compared to paperless, paperless-ng works a little different under the hood and has
more moving parts that work together. While this increases the complexity of
the system, it also brings many benefits.

Paperless consists of the following components:

*   **The webserver:** This is pretty much the same as in paperless. It serves
    the administration pages, the API, and the new frontend. This is the main
    tool you'll be using to interact with paperless. 

*   **The consumer:** This is what watches your consumption folder for documents.
    However, the consumer itself does not really consume your documents anymore.
    It rather notifies a task processor that a new file is ready for consumption.
    I suppose it should be named differently.
    This also used to check your emails, but that's now gone elsewhere as well.

*   **The task processor:** Paperless relies on [Django Q](https://django-q.readthedocs.io/en/latest/)
    for doing much of the heavy lifting. This is a task queue that accepts tasks from
    multiple sources and processes tasks in parallel. It also comes with a scheduler that executes
    certain commands periodically.

    This task processor is responsible for:

    *   Consuming documents. When the consumer finds new documents, it notifies the task processor to
        start a consumption task.
    *   Consuming emails. It periodically checks your configured accounts for new mails and
        produces consumption tasks for any documents it finds.
    *   The task processor also performs the consumption of any documents you upload through
        the web interface.
    *   Maintain the search index and the automatic matching algorithm. These are things that paperless
        needs to do from time to time in order to operate properly.

    This allows paperless to process multiple documents from your consumption folder in parallel! On
    a modern multi core system, consumption with full ocr is blazing fast.

    The task processor comes with a built-in admin interface that you can use to see whenever any of the
    tasks fail and inspect the errors (i.e., wrong email credentials, errors during consuming a specific
    file, etc).

*   A [redis](https://redis.io/) message broker: This is a really lightweight service that is responsible
    for getting the tasks from the webserver and consumer to the task scheduler. These run in different
    processes (maybe even on different machines!), and therefore, this is necessary.

*   Optional: A database server. Paperless supports both PostgreSQL and SQLite for storing its data.


## Installation
You can go multiple routes with setting up and running Paperless, but I chose the `docker` route, like so:

* The `docker route`

The `docker route` is quick and easy. This is the recommended route. This configures all the stuff
from above automatically so that it just works and uses sensible defaults for all configuration options.


### Docker Route

1.  Install `Docker` and `docker-compose`.

        Docker installation guide: https://docs.docker.com/engine/installation/
        docker-compose installation guide: https://docs.docker.com/compose/install/

2.  Copy either `docker-compose.sqlite.yml` or `docker-compose.postgres.yml` to
    `docker-compose.yml`, depending on which database backend you want to use.

        For new installations, it is recommended to use PostgreSQL as the database
        backend.

3.  Modify `docker-compose.yml` to your preferences. You may want to change the path
    to the consumption directory in this file. Find the line that specifies where
    to mount the consumption directory:

        - ./consume:/usr/src/paperless/consume

    Replace the part BEFORE the colon with a local directory of your choice:

        - /Volumes/Paperless/consume:/usr/src/paperless/consume

    Don't change the part after the colon or paperless wont find your documents.


3.  Modify `docker-compose.env`, following the comments in the file. The
    most important change is to set ``USERMAP_UID`` and ``USERMAP_GID``
    to the UID and GID of your user on the host system. This ensures that
    both the docker container and you on the host machine have write access
    to the consumption directory. If your UID and GID on the host system is
    1000 (the default for the first normal user on most systems), it will
    work out of the box without any modifications.

        You can use any settings from the file `paperless.conf` in this file.
        Have a look at `configuration` to see whats available.

4.  Run `docker-compose up -d`. This will create and start the necessary
    containers. This will also build the image of paperless if you grabbed the
    source archive.

5.  To be able to login, you will need a super user. To create it, execute the
    following command:

        $ docker-compose run --rm webserver createsuperuser

    This will prompt you to set a username, an optional e-mail address and
    finally a password.

6.  The default `docker-compose.yml` exports the webserver on your local port
    8000. If you haven't adapted this, you should now be able to visit your
    Paperless instance at `http://127.0.0.1:8000`. You can login with the
    user and password you just created.


