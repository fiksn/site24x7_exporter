# site24x7_exporter

[![GitHub Actions Workflow](https://github.com/svenstaro/site24x7_exporter/actions/workflows/ci.yml/badge.svg)](https://github.com/svenstaro/site24x7_exporter/actions)
[![Docker Hub](https://img.shields.io/docker/pulls/svenstaro/site24x7_exporter)](https://cloud.docker.com/repository/docker/svenstaro/site24x7_exporter/)
[![codecov](https://codecov.io/gh/svenstaro/site24x7_exporter/branch/master/graph/badge.svg)](https://codecov.io/gh/svenstaro/site24x7_exporter)
[![license](http://img.shields.io/badge/license-MIT-blue.svg)](https://github.com/svenstaro/site24x7_exporter/blob/master/LICENSE)
[![Lines of Code](https://tokei.rs/b1/github/svenstaro/site24x7_exporter)](https://github.com/svenstaro/site24x7_exporter)

A Prometheus compatible exporter for [site24x7.com](https://www.site24x7.com/)

## Features

This exporter currently supports these monitor types:

- URL ["Website"](https://www.site24x7.com/help/admin/adding-a-monitor/website-monitoring.html)
- HOMEPAGE ["Web Page Speed (Browser)"](https://www.site24x7.com/help/admin/adding-a-monitor/web-page-analyzer.html)
- RESTAPI ["REST API"](https://www.site24x7.com/help/admin/adding-a-monitor/rest-api-monitor.html)
- REALBROWSER ["Web Transaction (Browser)"](https://www.site24x7.com/help/admin/adding-a-monitor/webapplication-monitoring-realbrowser.html)

It also supports monitor groups and exposes them via tags.

There is a special path (default at `/geolocation`) which exposes geolocation information
with keys that reflect the names of the locations as provided by the site24x7 API.
This allows you to easily visualize locations on a map, for instance.
The list of locations is currently highly incomplete and only serves my purposes.
Pull requests welcome!

## CLI usage

```
A Prometheus compatible exporter for site24x7

Usage: site24x7_exporter [OPTIONS]

Options:
      --site24x7-endpoint <SITE24X7_ENDPOINT>
          API endpoint to use (depends on region, see https://site24x7.com/help/api) [default: site24x7.com]
          [possible values: site24x7.com, site24x7.eu, site24x7.cn, site24x7.in, site24x7.net.au]
      --web.listen-address <LISTEN_ADDRESS>
          Address on which to expose metrics and web interface [default: 0.0.0.0:9803]
      --web.telemetry-path <METRICS_PATH>
          Path under which to expose metrics [default: /metrics]
      --web.geolocation-path <GEOLOCATION_PATH>
          Path under which to expose geolocation information [default: /geolocation]
      --log.level <LOGLEVEL>
          Only log messages with the given severity or above [default: info]
  -h, --help
          Print help
  -V, --version
          Print version
```

## Using with proxies

If you need to use proxies in order to make the outgoing HTTP requests, you can set the environment variables
`https_proxy` and `HTTPS_PROXY` (the latter taking precedence). The proxy will then be used automatically.
You can see that a proxy will be used as the startup sequence will tell you so.

## How to use

### Preparation

First you need to create an OAuth 2.0 application as per https://www.site24x7.com/help/api/index.html#getting-started
For instance, go to https://api-console.zoho.eu (or your whatever your region's endpoint is) and then create a new
application of type "Self Client". Note the client's `Client ID` and `Client Secret`, you'll need them later.

Now it's time to get a refresh token. First, we'll need to generate a temporary code. In the "Generate Code" tab,
enter `Site24x7.Reports.Read` as the scope.
Choose a time duration of 10 minutes for the code and finally click "CREATE". You'll receive a temporary code.

In order to get your permanent refresh token, prepare a new file `curl-secrets` with these contents:

    client_id=your-client-id&
    client_secret=your-client-secret&
    code=your-temporary-code&
    grant_type=authorization_code

and then run this curl:

    curl https://accounts.zoho.eu/oauth/v2/token -X POST -d @curl-secrets

We use the `curl-secrets` file for security purposes so that your secrets won't be temporarily visible to all users
in a multiuser system.

Note: Remember to use your proper region endpoint!

You'll get back a response that looks roughly like this:

```
{
    "access_token": "some long token ",
    "api_domain": "https://www.zohoapis.eu",
    "expires_in": 3600,
    "refresh_token": "we're interested in whatever is in here",
    "token_type": "Bearer"
}
```

Copy the value of `refresh_token` somewhere, we'll need it later.

### Run via cargo

The exporter expects to receive the OAuth 2.0 data via environment variables.
These are:

- `ZOHO_CLIENT_ID`
- `ZOHO_CLIENT_SECRET`
- `ZOHO_REFRESH_TOKEN`

Let's set them and spin up our exporter:

    export ZOHO_CLIENT_ID=your-client-id
    export ZOHO_CLIENT_SECRET=your-client-secret
    export ZOHO_REFRESH_TOKEN=your-refresh-token
    cargo run -- --site24x7-endpoint site24x7.eu
    curl http://localhost:9803/metrics

Alternatively you can add these environment variables to an `.env` file in this format:

    ZOHO_CLIENT_ID=your-client-id
    ZOHO_CLIENT_SECRET=your-client-secret
    ZOHO_REFRESH_TOKEN=your-refresh-token

This is especially convenient for development purposes or local Docker usage as shown below.

### Run via docker

    docker run --env-file ./.env -p 9803:9803 svenstaro/site24x7_exporter --site24x7-endpoint site24x7.eu

### Testing

Try

    curl localhost:9803/metrics

and you should see some sweet metrics if everything is working fine.

### Troubleshooting

In case you get weird errors, try running with `--log.level debug` and then make a request
against the metrics endpoint. You should see a ton of helpful output. Be super careful though
as this **WILL EXPOSE SECRETS**. Do NOT run `--log.level debug` or `--log.level trace` for any
purposes except for local debugging.

## Usage in Prometheus

Make sure to not poll this too often as site24x7 has API usage limits per day.
The limit seems to be around 70000 per day so polling every 5 seconds should be safe.

## Running the tests

If you want to run the test suite, you'll need to run it as

    cargo test -- --test-threads 1 --nocapture

## Releasing

In order to release this, do:

- `cargo release <version>`
- `cargo release --execute <version>`
- Releases will automatically be deployed by GitHub Actions.
