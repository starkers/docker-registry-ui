## Docker Registry UI

[![Go Report Card](https://goreportcard.com/badge/github.com/quiq/docker-registry-ui)](https://goreportcard.com/report/github.com/quiq/docker-registry-ui)

### Overview

* Web UI for Docker Registry 2.6+
* Browse repositories and tags
* Display Docker image details by layers including both manifests v1 and v2
* Fast and small, written on Go
* Automatically discover an authentication method (basic auth, token service etc.)
* Caching the list of repositories, tag counts and refreshing in background
* Event listener of notification events coming from Registry
* Store events in sqlite or MySQL database
* CLI option to maintain the tags retention: purge tags older than X days keeping at least Y tags

No TLS or authentication implemented on the UI web server itself.
Assuming you will proxy it behind nginx, oauth2_proxy or something.

Docker images [quiq/docker-registry-ui](https://hub.docker.com/r/quiq/docker-registry-ui/tags/)

### Configuration

The configuration is stored in `config.yml` and the options are self-descriptive.

### Run UI

    docker run -d -p 8000:8000 -v /local/config.yml:/opt/config.yml:ro \
        --name=registry-ui quiq/docker-registry-ui

To run with your own root CA certificate, add to the command:

    -v /local/rootcacerts.crt:/etc/ssl/certs/ca-certificates.crt:ro

To preserve sqlite db file with event notifications data, add to the command:

    -v /local/data:/opt/data

You can also run the container with `--read-only` option, however when using using event listener functionality
you need to ensure the sqlite db can be written, i.e. mount a folder as listed above.

## Configure event listener on Docker Registry

To receive events you need to configure Registry as follow:

    notifications:
      endpoints:
        - name: docker-registry-ui
          url: http://docker-registry-ui.local:8000/api/events
          headers:
            Authorization: [Bearer abcdefghijklmnopqrstuvwxyz1234567890]
          timeout: 1s
          threshold: 5
          backoff: 10s
          ignoredmediatypes:
            - application/octet-stream

Adjust url and token as appropriate.
If you are running UI from non-root base path, e.g. /ui, the URL path for above will be `/ui/api/events`.

## Using MySQL instead of sqlite3 for event listener

To use MySQL as a storage you need to change `event_database_driver` and `event_database_location`
settings in the config file. It is expected you create a database mentioned in the location DSN.
Minimal privileges are `SELECT`, `INSERT`, `DELETE`.
You can create a table manually if you don't want to grant `CREATE` permission:

	CREATE TABLE events (
		id INTEGER PRIMARY KEY AUTO_INCREMENT,
		action CHAR(4) NULL,
		repository VARCHAR(100) NULL,
		tag VARCHAR(100) NULL,
		ip VARCHAR(15) NULL,
		user VARCHAR(50) NULL,
		created DATETIME NULL
	);

### Schedule a cron task for purging tags

To delete tags you need to enable the corresponding option in Docker Registry config. For example:

    storage:
      delete:
        enabled: true

The following example shows how to run a cron task to purge tags older than X days but also keep
at least Y tags no matter how old. Assuming container has been already running.

    10 3 * * * root docker exec -t registry-ui /opt/docker-registry-ui -purge-tags

You can try to run in dry-run mode first to see what is going to be purged:

    docker exec -t registry-ui /opt/docker-registry-ui -purge-tags -dry-run

Alternatively, you can schedule the purging task with built-in cron feature:

    purge_tags_keep_days: 90
    purge_tags_keep_count: 2
    purge_tags_schedule: '0 10 3 * * *'

Note, the cron schedule format includes seconds! See https://godoc.org/github.com/robfig/cron

### Debug mode

To increase http request verbosity, run container with `-e GOREQUEST_DEBUG=1`.

### Screenshots

![image](screenshots/1.png)

![image](screenshots/2.png)
