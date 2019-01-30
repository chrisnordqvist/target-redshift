# Target Redshift

[![CircleCI](https://circleci.com/gh/datamill-co/target-redshift.svg?style=svg)](https://circleci.com/gh/datamill-co/target-redshift)

[![PyPI version](https://badge.fury.io/py/target-redshift.svg)](https://pypi.org/project/target-redshift/)

[![](https://img.shields.io/librariesio/github/datamill-co/target-redshift.svg)](https://libraries.io/github/datamill-co/target-redshift)

A [Singer](https://singer.io/) redshift target, for use with Singer streams generated by Singer taps.

## Features

- Creates SQL tables for [Singer](https://singer.io) streams
- Denests objects flattening them into the parent object's table
- Denests rows into separate tables
- Adds columns and sub-tables as new fields are added to the stream [JSON Schema](https://json-schema.org/)
- Full stream replication via record `version` and `ACTIVATE_VERSION` messages.

## Install

```sh
pip install target-redshift
```

## Usage

1. Follow the
   [Singer.io Best Practices](https://github.com/singer-io/getting-started/blob/master/docs/RUNNING_AND_DEVELOPING.md#running-a-singer-tap-with-a-singer-target)
   for setting up separate `tap` and `target` virtualenvs to avoid version
   conflicts.

1. Create a [config file](#configjson) at
   `~/singer.io/target_redshift_config.json` with redshift connection
   information and target redshift schema.

   ```json
   {
     "redshift_host": "aws.something.or.other",
     "redshift_port": 5439,
     "redshift_database": "my_analytics",
     "redshift_username": "myuser",
     "redshift_password": "1234",
     "redshift_schema": "mytapname",
     "target_s3": {
       "aws_access_key_id": "AKIA...",
       "aws_secret_access_key": "supersecret",
       "bucket": "target_redshift_staging",
       "key_prefix": "__tmp"
     }
   }
   ```

1. Run `target-redshift` against a [Singer](https://singer.io) tap.

   ```bash
   ~/.virtualenvs/tap-something/bin/tap-something \
     | ~/.virtualenvs/target-redshift/bin/target-redshift \
       --config ~/singer.io/target_redshift_config.json
   ```

### Config.json

The fields available to be specified in the config file are specified
here.

| Field | Type | Default | Details |
| ----- | ---- | ------- | ------- |
| `redshift_host` |`["string"]` | `N/A` | |
| `redshift_port` | `["integer", "null"]`|  `5432` | |
| `redshift_database` | `["string"]`|  `N/A` | |
| `redshift_username` | `["string"]` |  `N/A` | |
| `redshift_password` | `["string"]`|  `N/A` | |
| `redshift_schema` | `["string", "null"]` |  `"public"` | |
| `invalid_records_detect` | `["boolean", "null"]`| `true` | Include `false` in your config to disable `target-redshift` from crashing on invalid records |
| `invalid_records_threshold` | `["integer", "null"]` | `0` | Include a positive value `n` in your config to allow for `target-redshift` to encounter at most `n` invalid records per stream before giving up. |
| `disable_collection` | `["string", "null"]` | `false` | Include `true` in your config to disable [Singer Usage Logging](#usage-logging). |
| `logging_level` | `["string", "null"]` | `"INFO"` | The level for logging. Set to `DEBUG` to get things like queries executed, timing of those queries, etc. See [Python's Logger Levels](https://docs.python.org/3/library/logging.html#levels) for information about valid values. |
| `target_s3` | `["object"]` | `N/A` | See `S3` below |

#### S3 Config.json

| Field | Type | Default | Details |
| ----- | ---- | ------- | ------- |
| `aws_access_key_id` |`["string"]` | `N/A` | |
| `aws_secret_access_key` |`["string"]` | `N/A` | |
| `bucket` |`["string"]` | `N/A` | Bucket where staging files should be uploaded to. |
| `key_prefix` |`["string", "null"]` | `""` | Prefix for staging file uploads to allow for better delineation of tmp files|

## Known Limitations

- Ignores `STATE` Singer messages.
- Requires a [JSON Schema](https://json-schema.org/) for every stream.
- Only string, string with date-time format, integer, number, boolean,
  object, and array types with or without null are supported. Arrays can
  have any of the other types listed, including objects as types within
  items.
    - Example of JSON Schema types that work
        - `['number']`
        - `['string']`
        - `['string', 'null']`
    - Exmaple of JSON Schema types that **DO NOT** work
        - `['string', 'integer']`
        - `['integer', 'number']`
        - `['any']`
        - `['null']`
- JSON Schema combinations such as `anyOf` and `allOf` are not supported.
- JSON Schema $ref is partially supported:
  - ***NOTE:*** The following limitations are known to **NOT** fail gracefully
  - Presently you cannot have any circular or recursive `$ref`s
  - `$ref`s must be present within the schema:
    - URI's do not work
    - if the `$ref` is broken, the behaviour is considered unexpected
- Any values which are the `string` `NULL` will be streamed to Redshift as the literal `null`
- Table names are restricted to:
  - 127 characters in length
  - can only be composed of `_`, lowercase letters, numbers, `$`
  - cannot start with `$`
  - ASCII characters
- Field/Column names are restricted to:
  - 127 characters in length
  - ASCII characters
- Fields/Columns are ***ALL*** `nullable`
- Fields/Columns use the default _largest_ type available for them

## Usage Logging

[Singer.io](https://www.singer.io/) requires official taps and targets to collect anonymous usage data. This data is only used in aggregate to report on individual tap/targets, as well as the Singer community at-large. IP addresses are recorded to detect unique tap/targets users but not shared with third-parties.

To disable anonymous data collection set `disable_collection` to `true` in the configuration JSON file.

## Developing

`target-redshift` utilizes [setup.py](https://python-packaging.readthedocs.io/en/latest/index.html) for package
management, and [PyTest](https://docs.pytest.org/en/latest/contents.html) for testing.

### Docker

If you have [Docker](https://www.docker.com/) and [Docker Compose](https://docs.docker.com/compose/) installed, you can
easily run the following to get a local env setup quickly.

First, make sure to create a [`.env`](https://docs.docker.com/compose/environment-variables/#the-env-file) file in the root of this repo (it has been `.gitignore`d so don't worry about accidentally staging it).

Therein, fill out the following information:

```sh
REDSHIFT_HOST='<your-host-name>' # Most likely 'localhost'
REDSHIFT_DATABASE='<your-db-name>' # Most likely 'dev'
REDSHIFT_SCHEMA='<your-schema-name>' # Probably 'public'
REDSHIFT_PORT='<your-port>' # Probably 5439
REDSHIFT_USERNAME='<your-user-name'
REDSHIFT_PASSWORD='<your-password>'
TARGET_S3_AWS_ACCESS_KEY_ID='<AKIA...>'
TARGET_S3_AWS_SECRET_ACCESS_KEY='<secret>'
TARGET_S3_BUCKET='<bucket-string>'
TARGET_S3_KEY_PREFIX='<some-string>' # We use 'target_redshift_test'
```

```sh
$ docker-compose up -d --build
$ docker logs -tf target-redshift_target-redshift_1 # You container names might differ
```

As soon as you see `INFO: Dev environment ready.` you can shell into the container and start running test commands:

```sh
$ docker exec -it target-redshift_target-redshift_1 bash # Your container names might differ
```

See the [PyTest](#pytest) commands below!

### DB

To run the tests, you will need an _actual_ Redshift cluster running.

Make sure to set the following env vars for [PyTest](#pytest):

```sh
$ EXPORT REDSHIFT_HOST='<your-host-name>' # Most likely 'localhost'
$ EXPORT REDSHIFT_DATABASE='<your-db-name>' # Most likely 'dev'
$ EXPORT REDSHIFT_SCHEMA='<your-schema-name>' # Probably 'public'
$ EXPORT REDSHIFT_PORT='<your-port>' # Probably 5439
$ EXPORT REDSHIFT_USERNAME='<your-user-name'
$ EXPORT REDSHIFT_PASSWORD='<your-password>' # Redshift requires passwords
```

### S3

To run the tests, you will need an _actual_ S3 bucket available.

Make sure to set the following env vars for [PyTest](#pytest):

```sh
$ EXPORT TARGET_S3_AWS_ACCESS_KEY_ID='<AKIA...>'
$ EXPORT TARGET_S3_AWS_SECRET_ACCESS_KEY='<secret>'
$ EXPORT TARGET_S3_BUCKET='<bucket-string>'
$ EXPORT TARGET_S3_KEY_PREFIX='<some-string>' # We use 'target_redshift_test'
```

### PyTest

To run tests, try:

```sh
$ python setup.py pytest
```

If you've `bash` shelled into the Docker Compose container ([see above](#docker)), you should be able to simply use:

```sh
$ pytest
```

## Sponsorship

Target Redshift is sponsored by Data Mill (Data Mill Services, LLC) [datamill.co](https://datamill.co/).

Data Mill helps organizations utilize modern data infrastructure and data science to power analytics, products, and services.

------
Copyright Data Mill Services, LLC 2018