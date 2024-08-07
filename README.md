# Wanderlog build instructions

This is a fork of winston-logstash, a plugin for our Winston logger to write
outputs to Logstash (part of the Kibana/ELK stack.)

## How to maintain

1.  Make changes and merge them into the `main` branch
2.  Figure out the number of the previous release by visiting
    https://github.com/wanderlog/winston-logstash/releases
    e.g., if the previous release was
    `v1.2.1-wanderlog.3`, change it to `v1.2.1-wanderlog.4`
3.  Run:
    ```shell
    VERSION='<substitute version here>' # e.g., VERSION='v1.2.1-wanderlog.4'
    VERSION_WITHOUT_V="${VERSION#v}"
    git checkout wanderlog-built
    git fetch
    git reset --hard origin/main
    yarn build
    # Increment the version number to the version of the latest release:
    # e.g., 1.2.1-wanderlog.4 from the example above
    # We use perl instead of sed so it's portable on Mac and non-Mac:
    # see https://stackoverflow.com/a/4247319/309011
    perl -i -pe's/"version":.*,/"version": "'"$VERSION_WITHOUT_V"'",/' package.json
    git add package.json lib
    git commit -m "[$VERSION] Release built version"
    git push -u origin wanderlog-built -f
    ```
4.  Visit https://github.com/wanderlog/winston-logstash/releases
5.  Create a new release with:
    - Title: new release (e.g., `v1.2.1-wanderlog.4`)
    - Branch: `wanderlog-built`

# winston-logstash

[![Build Status](https://github.com/jaakkos/winston-logstash/actions/workflows/build-test.yaml/badge.svg)](https://github.com/jaakkos/winston-logstash/actions/workflows/build-test.yaml)

[![Integration tests with Logstash instance](https://github.com/jaakkos/winston-logstash/actions/workflows/integration-test.yaml/badge.svg?branch=main)](https://github.com/jaakkos/winston-logstash/actions/workflows/integration-test.yaml)

A [Logstash TCP][0] transport for [winston][1].

## Usage

### Winston 2.x

``` js
// See test cases from test-bench/winston-2x/test/smoke.js
const winston = require("winston");
const transports = require("winston-logstash");

const logger = new winston.Logger({
  transports: [
    new transports.Logstash({
      port: 28777,
      node_name: "my node name",
      host: "127.0.0.1",
    }),
  ],
});

logger.info("Hello world!");
```

### Winston 3.x

``` js
// See test cases from test-bench/winston-3x/test/smoke.js
const winston = require("winston");
const LogstashTransport = require("winston-logstash/lib/winston-logstash-latest");

const logger = winston.createLogger({
  transports: [
    new LogstashTransport({
      port: 28777,
      node_name: "my node name",
      host: "127.0.0.1",
    }),
  ],
});

logger.info("Hello world!");
```

### Logstash config

``` ruby
# See example from test-bench/logstash/logstash/pipeline/default.conf
input {
  # Sample input over TCP
  tcp { port => 28777 type=>"sample" }
}
output {
  stdout { debug => true }
}

filter {
  json {
    source => "message"
  }
}
```

## FAQ

### Error handling with the transport

While this is a relatively complex transport with an internal state and IO, I think the current solution is the yet best approach to network failure. Transport is transparent about the errors and lets the user decide what to do in the case of an error. Suppose the developer chooses not to address them, it fallback to a safe default, an exception that stops the process. I think this way; it's simple but not always easy.

You can check the test case from `test-bench` folder where is the test case per Winston's version. The simplest ways to write the error logic would be:

#### Winston 2.x

For Winston 2.x you have to add the error listener to the transport.

``` js
const logstashTransport =  new LogstashTransport({...});

logstashTransport.on('error', (error) => {
  // Make the decission in here
  throw new Error('Stop the press, logging not working');
});

```

#### Winston 3.x

For Winston 3.x you have to add the error listener to the logger. Remember that there might be also other errors which are not originated from LogstashTransport.

``` js
const logger = winston.createLogger({
      transports: [
        new LogstashTransport({
              ...
               max_connect_retries: -1
              ...
              })]});

logger.on('error', (error) => {
  // Make the decission in here
  throw new Error('Stop the press, logging not working');
});
```

### What configuration options are available?

See documentation from [docs/configuration](docs/configuration.md)

## Run Tests

```shell
  npm test
```

## Run integration tests with Logstash

```shell
  cd test-bench/winston-3x
  docker-compose up -d
  npm test
```

## Inspiration

[winston-loggly][2]

## Author: [Jaakko Suutarla](https://github.com/jaakkos)

## License: MIT

See LICENSE for the full license text.

[0]: http://logstash.net/
[1]: https://github.com/flatiron/winston
[2]: https://github.com/indexzero/winston-loggly
