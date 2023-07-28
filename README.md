# `app-lblod-harvester`

Mu-semtech stack for harvesting and processing Decisions from external sources.
For the harvesting of Worship Decisions with focus on authentication, see
[`app-lblod-harvester-worship`](https://github.com/lblod/app-lblod-harvester-worship).

## List of Services

See the `docker-compose.yml` file.

## Setup and startup

To start this stack, clone this repository and start it using `docker compose`
using the following example snippet.

```bash
git clone git@github.com:lblod/app-lblod-harvester.git
cd app-lblod-harvester
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

It can take a while before everything is up and running. In case there is an
error the first time you launch, just stopping and relaunching `docker compose`
should resolve any issues:

```bash
docker-compose -f docker-compose.yml -f docker-compose.dev.yml stop
docker-compose -f docker-compose.yml -f docker-compose.dev.yml up -d
```

After starting, you might still have to wait for all the services to boot up,
and migrations to finish running. You can check this by inspecting the Docker
logs and wait for things to settle down. Once the stack is up and running
without errors you can visit the frontend in a browser on
`http://localhost:80`.

## Setting up the delta-producers

To ensure that the app can share data, it is necessary to set up the producers. We recommend that you first ensure a significant dataset has been harvested. The more data that has been harvested before setting up the producers, the faster the consumers will retrieve their data.

During its initial run, each producer performs a sync operation, which publishes the dataset as a `DCAT` dataset. This format can easily be ingested by consumers. After this initial sync, the producer switches to 'normal operation' mode, where it publishes delta files whenever new data is ingested.

Please note, for this app, we do not provide live delta-streaming. This means that delta files are not published immediately as new data gets ingested. Instead, delta files are created only during 'healing mode'. This is a background job that runs according to a specific cron pattern to create the delta files.
Check `./config/delta-producer/background-job-initiator/config.json` for the exact timings of the healing-job.

The reason why we do not provide live streaming is due to performance considerations. It has pushed us towards skipping `mu-authorization` and updating Virtuoso directly.

### Setting up producer besluiten

0. Only in case you are *fushing and restarting* from scratch, ensure

     ```json
        [
          {
            "name": "besluiten",
            # (...) other config

            "startInitialSync": false, # changed from 'true' to 'false'

            # (...) other config

          }
        ]
     ```
     - And also ensure some data has been harvested before starting the initial sync.

1. Make sure the app is up and running, and the migrations have run.
2. In `./config/delta-producer/background-job-initiator/config.json` file, make sure the following
   configuration is changed:

     ```json
        [
          {
            "name": "besluiten",
            # (...) other config

            "startInitialSync": true, # changed from 'false' to 'true'

            # (...) other config

          }
        ]
     ```
3. Restart the services: `drc restart delta-producer-background-jobs-initiator`
4. You can follow the status of the job, through the dashboard frontend.

### Triggering the healing-job manually
In some cases, you might want to trigger the healing job manually.
```
drc exec delta-producer-background-jobs-initiator wget --post-data='' http://localhost/besluiten/healing-jobs
```

## Additional notes

### Performance

The default Virtuoso settings might be too weak if you need to ingest the
production data. There is a better config for this that you can use in your
`docker-compose.override.yml`

```yaml
virtuoso:
  volumes:
    - ./data/db:/data
    - ./config/virtuoso/virtuoso-production.ini:/data/virtuoso.ini
    - ./config/virtuoso/:/opt/virtuoso-scripts
```

### `delta-producer-report-generator`

Not all required parameters are provided, since these are deploy specific, see
[the delta-producer-report-generator
repository](https://github.com/lblod/delta-producer-report-generator).

### `deliver-email-service`

Should have credentials provided, see [the deliver-email-service
repository](https://github.com/redpencilio/deliver-email-service).
