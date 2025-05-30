# repository-service-tuf-worker

Repository Service for TUF Worker

repository-service-tuf-worker is part of Repository Service for TUF (RSTUF)

## Getting Started

These instructions will cover usage information and for the docker container.

## Prerequisities


In order to run this container you'll need docker installed.

Other requirements include:

* repository-service-tuf-api service
* Compatible broker and result backend service with
  [Celery](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/index.html).
  Recomended: [RabbitMQ](https://www.rabbitmq.com) or [Redis](https://redis.com)
* PostgreSQL server
* Online key (see Online Key details below)

### Online Key
The RSTUF Worker requires an online key which is used for signing the TUF
metadata when an artifact is added or removed.

Here are some things you need to know:
* The key must be compatible with
  [Secure Systems Library](https://github.com/secure-systems-lab/securesystemslib).

* This key must be the same one used during the [RSTUF CLI ceremony](https://repository-service-tuf.readthedocs.io/en/latest/guide/repository-service-tuf-cli/index.html#ceremony-ceremony).

For more information read the [Deployment documentation](https://repository-service-tuf.readthedocs.io/en/latest/guide/deployment/index.html).

## Usage

### Container Parameters

```shell
docker run --env="RSTUF_STORAGE_BACKEND=LocalStorage" \
    --env="RSTUF_LOCAL_STORAGE_BACKEND_PATH=/metadata" \
    --env="RSTUF_BROKER_SERVER=guest:guest@rabbitmq:5672" \
    --env="RSTUF_REDIS_SERVER=redis://redis" \
    --env="RSTUF_DB_SERVER=postgresql://postgres:secret@postgres:5432" \
    ghcr.io/repository-service-tuf/repository-service-tuf-worker:latest \
```


### Environment Variables

#### (Required) `RSTUF_BROKER_SERVER`

Broker server address.

The broker must to be compatible with Celery.
See [Celery Broker Instructions](https://docs.celeryq.dev/en/stable/getting-started/backends-and-brokers/index.html#broker-instructions)

Example: `guest:guest@rabbitmq:5672`

#### (Required) `RSTUF_REDIS_SERVER`

Redis server address.

The result backend must to be compatible with Celery. See
[Celery Task result backend settings](https://docs.celeryq.dev/en/stable/userguide/configuration.html#task-result-backend-settings)

Example: `redis://redis`

#### (Required) `RSTUF_DB_SERVER`

RSTUF requires [PostgreSQL](https://www.postgresql.org)

> **Note:** [MySQL/MariaDB](https://mariadb.org) is also supported, but is not recommended
> for production use at this time. We still have an opened issue to solve.
> See https://github.com/repository-service-tuf/repository-service-tuf-worker/issues/598


Scheme
  - PostgreSQL: `postgresql://`
  - MySQL/MariaDB: `mysql+pymysql://` (not recommended for production)

Example: `postgresql://mypgsql:5432`

* Optional variables:

  * `RSTUF_DB_USER` optional information about the user name

    If using this optional variable:
    - Do not include the user in the `RSTUF_DB_SERVER`.
    - The `RSTUF_DB_PASSWORD` becomes required

  * `RSTUF_DB_PASSWORD` use this variable to provide the password separately.
    - Do not include the password in the `RSTUF_DB_SERVER`
    - This environment variable supports container secrets when the `/run/secrets`
      volume is added to the path.

  Example:
  ```
  RSTUF_DB_SERVER=postgresql://mypgsql:5432
  RSTUF_DB_USER=postgres
  RSTUF_DB_PASSWORD=/run/secrets/POSTGRES_PASSWORD
  ```

#### (Optional) `RSTUF_REDIS_SERVER_PORT`

Redis Server port number. Default: 6379

#### (Optional) `RSTUF_REDIS_SERVER_DB_RESULT`

Redis Server DB number for Result Backend (tasks). Default: 0

Important: It should use the same db id as used by RSTUF API.

#### (Optional) `RSTUF_BUMP_ONLINE_ROLES_TTL`

Bump Online Roles lock time to lease (TTL) in seconds. Default: 600

This tasks runs every 10 minutes. This is the TTL time in case a task gets
longer.
It avoid one or more RSTUF Workers running at the same task.

#### (Optional) `RSTUF_BUMP_ONLINE_ROLES_CHUNK_SIZE`

Define the bump online roles chunk size. Default: 500

RSTUF Worker can have large number of delegations (ex: hash bins).

RSTUF Worker calculates automatically the chunk. Depending on the
number of Workers and Roles it can be customized.

#### (Optional) `RSTUF_REDIS_SERVER_DB_REPO_SETTINGS`

Redis Server DB number for repository settings. Default: 1

This settings are shared accress the Repository Workers
(``repository-service-tuf-worker``) to have dynamic configuration.

#### (Required) `RSTUF_STORAGE_BACKEND`

Select a supported type of Storage Service.

Available types:

* `LocalStorage` (container volume)
* `AWSS3` (AWS S3)

##### `LocalStorage` (container volume)

* (Required) ``RSTUF_LOCAL_STORAGE_BACKEND_PATH``

  - Define the directory where the data will be saved, example: `storage`

##### `AWSS3` (AWS S3)

* (Required) ``RSTUF_AWS_STORAGE_BUCKET``

  The name of s3 bucket to use.

**_NOTE:_** It requires the AWS credentials to be set in the environment variables.
See the AWS3 Environment Variables section below.

**_NOTE:_**  The AWS3 supports all `boto3`
[environment variables](https://boto3.amazonaws.com/v1/documentation/api/latest/guide/configuration.html#using-environment-variables).


#### (Optional) AWS Environment Variables
*  ``RSTUF_AWS_ACCESS_KEY_ID``

  The access key to use when creating the client session to the S3.

  This environment variable supports container secrets when the ``/run/secrets``
  volume is added to the path.
  Example: `RSTUF_AWS_ACCESS_KEY_ID=/run/secrets/S3_ACCESS_KEY`

*  ``RSTUF_AWS_SECRET_ACCESS_KEY``

  The secret key to use when creating the client session to the S3.

  This environment variable supports container secrets when the ``/run/secrets``
  volume is added to the path.
  Example: ``RSTUF_AWS_SECRET_ACCESS_KEY=/run/secrets/S3_SECRET_KEY``

*  ``RSTUF_AWS_DEFAULT_REGION``

  The name of the region associated with the S3.

*  ``RSTUF_AWS_ENDPOINT_URL``

  The complete URL to use for the constructed client. Normally, the
  client automatically constructs the appropriate URL to use when
  communicating with a service.

*  ``RSTUF_AWS_S3_OBJECT_ACL``

  Define the ACL creation to the JSON TUF Metadata in the S3 bucket.

  Default is ``public-read``. See [Boto3 documentation](https://boto3.amazonaws.com/v1/documentation/api/latest/reference/services/s3/client/put_object_acl.html)
  for all supported Options.

#### (Optional) Google Cloud Environment Variables

*  ``RSTUF_GOOGLE_APPLICATION_CREDENTIALS``

  The path to the Google Cloud credentials file.

  Example: `RSTUF_GOOGLE_APPLICATION_CREDENTIALS=/run/secrets/google-credentials.json`

#### (Optional) `RSTUF_LOCK_TIMEOUT`

Timeout for publishing JSON metadata files. Default: 60.0 (seconds)

This timeout avoids race conditions leading to publishing JSON metadata files by multiple
Worker's services. It guarantees that the metadata is consistent in the backend
storage service (`RSTUF_STORAGE_BACKEND`).

In most use cases, the timeout of 60.0 seconds is sufficient.

#### (Optional) `RSTUF_ONLINE_KEY_DIR`

Directory path for online signing key file. Expected file format is unencrypted PKCS8/PEM.

Make sure to use the secrets management service of your deployment platform to
protect the private key!

Example:
- `RSTUF_ONLINE_KEY_DIR=/run/secrets`
- RSTUF worker expects related private key under  `/run/secrets/<file name>`


#### (Optional) `RSTUF_WORKER_ID`

Custom Worker ID.  Default: `hostname` (Container hostname)

#### (Optional) `DATA_DIR`

Container data directory. Default: `/data`


### Persistent data

* `$DATA_DIR`. Default: `/data`

### Customization/Tuning

The `repository-service-tuf-worker` uses supervisord and uses a
`supervisor.conf` from `$DATA_DIR`.

It can be used to customize/tuning performance of Celery.
