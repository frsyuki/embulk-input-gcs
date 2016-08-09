# Google Cloud Storage file input plugin for Embulk
[![Build Status](https://travis-ci.org/embulk/embulk-input-gcs.svg?branch=master)](https://travis-ci.org/embulk/embulk-input-gcs)

## Overview

* Plugin type: **file input**
* Resume supported: **yes**
* Cleanup supported: **yes**

## Usage

### Install plugin

```
embulk gem install embulk-input-gcs
```

### Google Service Account Settings

If you chose "private_key" or "json_key" as [auth_method](#Authentication), you can get service_account_email and private_key or json_key like below.

1. Make project at [Google Developers Console](https://console.developers.google.com/project).

1. Make "Service Account" with [this step](https://cloud.google.com/storage/docs/authentication#service_accounts).

    Service Account has two specific scopes: read-only, read-write.

    embulk-input-gcs can run "read-only" scopes.

1. Generate private key in P12(PKCS12) format or json_key, and upload to machine.

### run

```
embulk run /path/to/config.yml
```

## Configuration

- **bucket** Google Cloud Storage bucket name (string, required)
- **path_prefix** prefix of target keys (string, either of "path_prefix" or "paths" is required)
- **paths** list of target keys (array of string, either of "path_prefix" or "paths" is required)
- **incremental**: enables incremental loading(boolean, optional. default: true. If incremental loading is enabled, config diff for the next execution will include `last_path` parameter so that next execution skips files before the path. Otherwise, `last_path` will not be included.
- **auth_method**  (string, optional, "private_key", "json_key" or "compute_engine". default value is "private_key")
- **service_account_email** Google Cloud Storage service_account_email (string, required when auth_method is private_key)
- **p12_keyfile** fullpath of p12 key (string, required when auth_method is private_key)
- **json_keyfile** fullpath of json_key (string, required when auth_method is json_key)
- **application_name** application name anything you like (string, optional)

## Example

```yaml
in:
  type: gcs
  bucket: my-gcs-bucket
  path_prefix: logs/csv-
  auth_method: private_key #default
  service_account_email: ABCXYZ123ABCXYZ123.gserviceaccount.com
  p12_keyfile: /path/to/p12_keyfile.p12
  application_name: Anything you like
```

Example for "sample_01.csv.gz" , generated by [embulk example](https://github.com/embulk/embulk#trying-examples)

```yaml
in:
  type: gcs
  bucket: my-gcs-bucket
  path_prefix: sample_
  auth_method: private_key #default
  service_account_email: ABCXYZ123ABCXYZ123.gserviceaccount.com
  p12_keyfile: /path/to/p12_keyfile.p12
  application_name: Anything you like
  decoders:
  - {type: gzip}
  parser:
    charset: UTF-8
    newline: CRLF
    type: csv
    delimiter: ','
    quote: '"'
    header_line: true
    columns:
    - {name: id, type: long}
    - {name: account, type: long}
    - {name: time, type: timestamp, format: '%Y-%m-%d %H:%M:%S'}
    - {name: purchase, type: timestamp, format: '%Y%m%d'}
    - {name: comment, type: string}
out: {type: stdout}
```

## Authentication

There are three methods supported to fetch access token for the service account.

1. Public-Private key pair of GCP(Google Cloud Platform)'s service account
2. JSON key of GCP(Google Cloud Platform)'s service account
3. Pre-defined access token (Google Compute Engine only)

### Public-Private key pair of GCP's service account

You first need to create a service account (client ID), download its private key and deploy the key with embulk.

```yaml
in:
  type: gcs
  auth_method: private_key
  service_account_email: ABCXYZ123ABCXYZ123.gserviceaccount.com
  p12_keyfile: /path/to/p12_keyfile.p12
```

### JSON key of GCP's service account

You first need to create a service account (client ID), download its json key and deploy the key with embulk.

```yaml
in:
  type: gcs
  auth_method: json_key
  json_keyfile: /path/to/json_keyfile.json
```

You can also embed contents of json_keyfile at config.yml.

```yaml
in:
  type: gcs
  auth_method: json_key
  json_keyfile:
    content: |
      {
          "private_key_id": "123456789",
          "private_key": "-----BEGIN PRIVATE KEY-----\nABCDEF",
          "client_email": "..."
       }
```

### Pre-defined access token(GCE only)

On the other hand, you don't need to explicitly create a service account for embulk when you
run embulk in Google Compute Engine. In this third authentication method, you need to
add the API scope "https://www.googleapis.com/auth/devstorage.read_only" to the scope list of your
Compute Engine VM instance, then you can configure embulk like this.

[Setting the scope of service account access for instances](https://cloud.google.com/compute/docs/authentication)

```yaml
in:
  type: gcs
  auth_method: compute_engine
```

## Eventually Consistency

An operation listing objects is eventually consistent although getting objects is strongly consistent, see https://cloud.google.com/storage/docs/consistency.

`path_prefix` uses the objects list API, therefore it would miss some of objects.
If you want to avoid such situations, you should use `paths` option which directly specifies object paths without the objects list API.

## Build

```
./gradlew gem
```

## Test

To run unit tests, we need to configure the following environment variables.

Additionally, following files will be needed to upload to existing GCS bucket.
* [sample_01.csv](./src/test/resources/sample_01.csv)
* [sample_02.csv](./src/test/resources/sample_02.csv)

When environment variables are not set, skip some test cases.

```
GCP_EMAIL
GCP_P12_KEYFILE
GCP_JSON_KEYFILE
GCP_BUCKET
GCP_BUCKET_DIRECTORY(optional, if needed)
```

If you're using Mac OS X El Capitan and GUI Applications(IDE), like as follows.
```
$ vi ~/Library/LaunchAgents/environment.plist
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
  <key>Label</key>
  <string>my.startup</string>
  <key>ProgramArguments</key>
  <array>
    <string>sh</string>
    <string>-c</string>
    <string>
      launchctl setenv GCP_EMAIL ABCXYZ123ABCXYZ123.gserviceaccount.com
      launchctl setenv GCP_P12_KEYFILE /path/to/p12_keyfile.p12
      launchctl setenv GCP_JSON_KEYFILE /path/to/json_keyfile.json
      launchctl setenv GCP_BUCKET my-bucket
      launchctl setenv GCP_BUCKET_DIRECTORY unittests
    </string>
  </array>
  <key>RunAtLoad</key>
  <true/>
</dict>
</plist>

$ launchctl load ~/Library/LaunchAgents/environment.plist
$ launchctl getenv GCP_EMAIL //try to get value.

Then start your applications.
```
