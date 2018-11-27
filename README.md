# docker-volume-backup

Docker image for performing simple backups of Docker volumes. Main features:

- Mount volumes into the container, and they'll get backed up
- Use full `cron` expressions for scheduling the backups
- Backs up to local disk, [AWS S3](https://aws.amazon.com/s3/), or both
- Optionally stops other containers for the duration of the backup, and starts them again afterward, to ensure consistent backups of things like database files, etc
- Optionally ships backup metrics to [InfluxDB](https://docs.influxdata.com/influxdb/), for monitoring

## Examples

### Backing up locally

Say you're running some dashboards with [Grafana](https://grafana.com/) and want to back them up:

```yml
version: "3"

services:

  dashboard:
    image: grafana/grafana:5.3.4
    volumes:
      - grafana-data:/var/lib/grafana           # This is where Grafana keeps its data

  backup:
    image: futurice/docker-volume-backup:1.1.0
    volumes:
      - grafana-data:/backup/grafana-data:ro    # Mount the Grafana data volume (as read-only)
      - ./backups:/archive                      # Mount a local folder as the backup archive

volumes:
  grafana-data:
```

This will back up the Grafana data volume, once per day, and write it to `./backups` with a filename like `backup-2018-11-26.tar.gz`.

### Backing up to S3

Off-site backups are better, though:

```yml
version: "3"

services:

  dashboard:
    image: grafana/grafana:5.3.4
    volumes:
      - grafana-data:/var/lib/grafana           # This is where Grafana keeps its data

  backup:
    image: futurice/docker-volume-backup:1.1.0
    environment:
      AWS_S3_BUCKET_NAME: my-backup-bucket      # S3 bucket which you own, and already exists
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}   # Read AWS secrets from environment (or a .env file)
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    volumes:
      - grafana-data:/backup/grafana-data:ro    # Mount the Grafana data volume (as read-only)

volumes:
  grafana-data:
```

This configuration will back up to AWS S3 instead.

### Stopping containers while backing up

It's not generally safe to read files to which other processes might be writing. You may end up with corrupted copies. You generally don't want corrupted backups.

You can give the backup container access to the Docker socket, and label any containers that need to be stopped while the backup runs:

```yml
version: "3"

services:

  dashboard:
    image: grafana/grafana:5.3.4
    volumes:
      - grafana-data:/var/lib/grafana           # This is where Grafana keeps its data
    labels:
      - "docker-volume-backup.stop-during-backup=true"

  backup:
    image: futurice/docker-volume-backup:1.1.0
    environment:
      AWS_S3_BUCKET_NAME: my-backup-bucket      # S3 bucket which you own, and already exists
      AWS_ACCESS_KEY_ID: ${AWS_ACCESS_KEY_ID}   # Read AWS secrets from environment (or a .env file)
      AWS_SECRET_ACCESS_KEY: ${AWS_SECRET_ACCESS_KEY}
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro # Allow use of the "stop-during-backup" feature
      - grafana-data:/backup/grafana-data:ro    # Mount the Grafana data volume (as read-only)

volumes:
  grafana-data:
```

This configuration allows you to safely back up things like databases, if you can tolerate a bit of downtime.

## Configuration

Variable | Default | Notes
--- | --- | ---
`BACKUP_SOURCES` | `/backup` | Where to read data from. This can be a space-separated list if you need to back up multiple paths, when mounting multiple volumes for example. On the other hand, you can also just mount multiple volumes under `/backup` to have all of them backed up.
`BACKUP_CRON_EXPRESSION` | `@daily` | Standard debian-flavored `cron` expression for when the backup should run. Use e.g. `0 4 * * *` to back up at 4 AM every night. See the [man page](http://man7.org/linux/man-pages/man8/cron.8.html) or [crontab.guru](https://crontab.guru/) for more.
`BACKUP_FILENAME` | `backup-%Y-%m-%d.tar.gz` | File name template for the backup file. Is passed through `date` for formatting. For example, `%Y-%m-%d-%H-%M-%S.tar.gz` would produce files that look like `2018-11-27-12-01-14.tar.gz`. See the [man page](http://man7.org/linux/man-pages/man1/date.1.html) for more.
`BACKUP_ARCHIVE` | `/archive` | When this path is available within the container (i.e. you've mounted a Docker volume there), a finished backup file will get archived there after each run.
`BACKUP_WAIT_SECONDS` | `0` | The backup script will sleep this many seconds between re-starting stopped containers, and proceeding with archiving/uploading the backup. This can be useful if you don't want the load/network spike of a large upload immediately after the load/network spike of container startup.
`BACKUP_HOSTNAME` | `$(hostname)` | Name of the host (i.e. Docker container) in which the backup runs. Mostly useful if you want a specific hostname to be associated with backup metrics (see InfluxDB support).
`DOCKER_STOP_OPT_IN_LABEL` | `docker-volume-backup.stop-during-backup` | Adding this label with the value `true` to other containers causes the backup script to stop them before the backup runs, and starting them back up after the backup has ran. This will only work if you also expose the Docker daemon via your `/var/run/docker.sock` into the backup container. See examples.
`AWS_S3_BUCKET_NAME` |  | When provided, the resulting backup file will be uploaded to this S3 bucket after the backup has ran.
`AWS_ACCESS_KEY_ID` |  | Required when using `AWS_S3_BUCKET_NAME`.
`AWS_SECRET_ACCESS_KEY` |  | Required when using `AWS_S3_BUCKET_NAME`.
`AWS_DEFAULT_REGION` |  | Optional when using `AWS_S3_BUCKET_NAME`. Allows you to override the AWS CLI default region. Usually not needed.
`INFLUXDB_URL` |  | When provided, backup metrics will be sent to an InfluxDB instance at this URL, e.g. `https://influxdb.example.com`.
`INFLUXDB_DB` |  | Required when using `INFLUXDB_URL`; e.g. `my_database`.
`INFLUXDB_CREDENTIALS` |  | Required when using `INFLUXDB_URL`; e.g. `user:pass`.
`INFLUXDB_MEASUREMENT` | `docker_volume_backup` | Required when using `INFLUXDB_URL`.

## Testing

A bunch of test cases exist under [`test`](test/). To run them:

    cd test/backing-up-locally/
    docker-compose stop && docker-compose rm -f && docker-compose build && docker-compose up

Some cases may need secrets available in the environment, e.g. for S3 uploads to work.

## Metrics

After the backup, the script will collect some metrics from the run. By default, they're just written out as logs. For example:

```
docker_volume_backup
host=my-demo-host
size_compressed_bytes=219984
containers_total=4
containers_stopped=1
time_wall=61.6939337253571
time_total=1.69393372535706
time_compress=0.171068429946899
time_upload=0.56016993522644
```

If so configured, they can also be shipped to an InfluxDB instance. This allows you to set up monitoring and/or alerts for them. Here's a sample visualization on Grafana:

![Backup dashboard sample](doc/backup-dashboard-sample.png)
