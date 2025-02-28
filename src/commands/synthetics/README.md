# Synthetics command

Run Synthetics tests from your CI.

## Usage

### Setup

You need to have a Datadog API key (`DATADOG_API_KEY`) and application key(`DATADOG_APP_KEY`) available in your environment or pass them to the CLI.

```bash
# Environment setup
export DATADOG_API_KEY="<API KEY>"
export DATADOG_APP_KEY="<APPLICATION KEY>"

# Passing to CLI
yarn datadog-ci synthetics <command> --apiKey "<API KEY>" --appKey "<APPLICATION KEY>"
```

It is possible to configure the tool to use other Datadog sites by defining the `DATADOG_SITE` environment variable. By default the requests are sent to Datadog US1.

If the org uses a custom sub-domain to access Datadog app, it needs to be set in the `DATADOG_SUBDOMAIN` environment variable or in the global configuration file under the `subdomain` key to properly display the test results URL. As an example, if the URL used to access Datadog is `myorg.datadoghq.com` then set the environment variable to `myorg`, ie:

```bash
export DATADOG_SUBDOMAIN="myorg"
```

You can use the `DATADOG_SYNTHETICS_LOCATIONS` to override the locations where your tests run. Locations should be separated with `;`. Note that the configuration in test files takes precedence over others overrides.

```bash
export DATADOG_SYNTHETICS_LOCATIONS="aws:us-east-1;aws:us-east-2"
```

### API

By default it runs at the root of the working directory and finds `{,!(node_modules)/**/}*.synthetics.json` files (every files ending with `.synthetics.json` except those in the `node_modules` folder).

#### Configuration

Configuration is done via a json file, by default the tool load `datadog-ci.json` which can be overridden through the `--config` argument.

The configuration file structure is the following:

```json
{
  "apiKey": "<DATADOG_API_KEY>",
  "appKey": "<DATADOG_APPLICATION_KEY>",
  "datadogSite": "datadoghq.com",
  "failOnCriticalErrors": true,
  "failOnMissingTests": true,
  "failOnTimeout": true,
  "files": "{,!(node_modules)/**/}*.synthetics.json",
  "global": {
    "allowInsecureCertificates": true,
    "basicAuth": {"username": "test", "password": "test"},
    "body": "{\"fakeContent\":true}",
    "bodyType": "application/json",
    "cookies": "name1=value1;name2=value2;",
    "defaultStepTimeout": 15,
    "deviceIds": ["chrome.laptop_large"],
    "executionRule": "skipped",
    "followRedirects": true,
    "headers": {"NEW_HEADER": "NEW VALUE"},
    "locations": ["aws:us-east-1"],
    "retry": {"count": 2, "interval": 300},
    "startUrl": "{{URL}}?static_hash={{STATIC_HASH}}",
    "startUrlSubstitutionRegex": "s/(https://www.)(.*)/$1extra-$2/",
    "variables": {"MY_VARIABLE": "new title"}
  },
  "pollingTimeout": 120000,
  "proxy": {
    "auth": {
      "username": "login",
      "password": "pwd"
    },
    "host": "127.0.0.1",
    "port": 3128,
    "protocol": "http"
  },
  "subdomain": "subdomainname"
}
```

**Proxy configuration**

It is possible to configure a proxy to be used for outgoing connections to Datadog using the `proxy` key of the global configuration file.

As the [`proxy-agent`](https://github.com/TooTallNate/node-proxy-agent) library is used to configure the proxy, protocols supported are `http, https, socks, socks4, socks4a, socks5, socks5h, pac+data, pac+file, pac+ftp, pac+http, pac+https`. The `proxy` key of the global configuration file is passed to a new `proxy-agent` instance, meaning same configuration than the library is supported.

**Note**: `host` and `port` keys are mandatory arguments and the `protocol` key defaults to `http` if not defined.

#### Sub-commands

The available sub-command is:

- `run-tests`: run the tests discovered in the folder according to the `files` configuration key

It accepts the `--public-id` (or shorthand `-p`) argument to trigger only the specified test. It can be set multiple times to run multiple tests:

```bash
yarn datadog-ci synthetics run-tests --public-id pub-lic-id1 --public-id pub-lic-id2
```

It is also possible to trigger tests corresponding to a search query by using the flag `--search` (or shorthand `-s`). With this option, the global configuration overrides applies to all tests discovered with the search query.

```bash
yarn datadog-ci synthetics run-tests -s 'tag:e2e-tests' --config global.config.json
```

You can use `--files` (shorthand `-f`) to override the global file selector.
It's particularly useful when you want to run multiple suites in parallel with a single global configuration file.

```bash
yarn datadog-ci synthetics run-tests -f ./component-1/**/*.synthetics.json -f ./component-2/**/*.synthetics.json
```

Variables can also be passed as arguments using `--variable KEY=VALUE`.

```bash
yarn datadog-ci synthetics run-tests -f ./component-1/**/*.synthetics.json -v PASSWORD=$PASSWORD
```

#### Failure modes flags

- `--failOnTimeout` (or `--no-failOnTimeout`) will make the CI fail (or pass) if one of the result exceed its test timeout.
- `--failOnCriticalErrors` will make the CI fail if tests were not triggered or results could not be fetched.
- `--failOnMissingTests` will make the CI fail if at least one test is missing.

### Test files

Your test files must be named with a `.synthetics.json` suffix.

```json
// myTest.synthetics.json
{
  "tests": [
    {
      "id": "<TEST_PUBLIC_ID>",
      "config": {
        "allowInsecureCertificates": true,
        "basicAuth": {"username": "test", "password": "test"},
        "body": "{\"fakeContent\":true}",
        "bodyType": "application/json",
        "cookies": "name1=value1;name2=value2;",
        "defaultStepTimeout": 15,
        "deviceIds": ["chrome.laptop_large"],
        "executionRule": "skipped",
        "followRedirects": true,
        "headers": {"NEW_HEADER": "NEW VALUE"},
        "locations": ["aws:us-east-1"],
        "pollingTimeout": 30000,
        "retry": {"count": 2, "interval": 300},
        "startUrl": "{{URL}}?static_hash={{STATIC_HASH}}",
        "startUrlSubstitutionRegex": "s/(https://www.)(.*)/$1extra-$2/",
        "variables": {"MY_VARIABLE": "new title"}
      }
    }
  ]
}
```

The `<TEST_PUBLIC_ID>` can be either the identifier of the test found in the URL of a test details page (eg. for `https://app.datadoghq.com/synthetics/details/abc-def-ghi` it would be `abc-def-ghi`) or the full URL to the details page (ie. directly `https://app.datadoghq.com/synthetics/details/abc-def-ghi`).

All options under the `config` key are optional and allow overriding the configuration of the test as stored in Datadog.

- `allowInsecureCertificates`: (boolean) disable certificate checks in API tests.
- `basicAuth`: (object) credentials to provide in case a basic authentication is encountered.
  - `username`: (string) username to use in basic authentication.
  - `password`: (string) password to use in basic authentication.
- `body`: (string) data to send in a synthetics API test.
- `bodyType`: (string) type of the data sent in a synthetics API test.
- `cookies`: (string or object) use provided string as Cookie header in API or Browser test, in addition or in replacement.
  - if a string, it will be used to replace the original cookies.
  - if an object, its format must be `{append?: boolean, value: string}` and depending on the value of `append`, it will be appended or will replace the original cookies.
- `defaultStepTimeout`: (number) maximum duration of steps in seconds for Browser tests, does not override individually set step timeouts.
- `deviceIds`: (array) list of devices on which to run the Browser test.
- `executionRule`: (string) execution rule of the test: it defines the behavior of the CLI in case of a failing test, it can be either:
  - `blocking`: the CLI returns an error if the test fails.
  - `non_blocking`: the CLI only prints a warning if the test fails.
  - `skipped`: the test is not executed at all.
- `followRedirects`: (boolean) indicates whether to follow or not HTTP redirections in API tests.
- `headers`: (object) headers to replace in the test. This object should contain as keys the name of the header to replace and as values the new value of the header.
- `locations`: (array) list of locations from which the test should be run.
- `pollingTimeout`: (integer) maximum duration in milliseconds of a test, if execution exceeds this value it is considered failed.
- `retry`: (object) retry policy for the test.
  - `count`: (integer) number of attempts to perform in case of test failure.
  - `interval`: (integer) interval between the attempts (in milliseconds).
- `startUrl`: (string) new start URL to provide to the test. Variables specified in brackets (`{{ EXAMPLE }}`) found in environment variables are replaced.
- `startUrlSubstitutionRegex`: (string) regex to modify the starting URL of the test (browser and HTTP tests only), whether it was given by the original test or by the configuration override `startUrl`. If the URL contains variables, this regex will be applied after the interpolation of the variables. The format is `s/your_regex/your_substitution/modifiers` and follow Javascript regex syntax, for instance `s/(https://www.)(.*)/$1extra-$2/` to transform `https://www.example.com` into `https://www.extra-example.com`.
- `variables`: (object) variables to replace in the test. This object should contain as keys the name of the variable to replace and as values the new value of the variable.

### Testing tunnel

You can run tests within your development environment by combining variable overrides with the [Testing Tunnel](https://docs.datadoghq.com/synthetics/testing_tunnel/#pagetitle). This allows you to run end-to-end encryption at every stage of your software development lifecycle, from pre-production environments through to your production system.

### End-to-end testing process

To verify this command works as expected, you can trigger a test run and verify it returns 0:

```bash
export DATADOG_API_KEY='<API key>'
export DATADOG_APP_KEY='<application key>'

yarn datadog-ci synthetics run-tests --public-id abc-def-ghi
```

Successful output should look like this:

```bash
[abc-def-ghi] Trigger test "Check on testing.website"
[abc-def-ghi] Waiting results for "Check on testing.website"


=== REPORT ===
Took 11546ms

✓ [abc-def-ghi] | Check on testing.website
  ✓ location: Frankfurt (AWS)
    ⎋  total duration: 28.9 ms - result url: https://app.datadoghq.com/synthetics/details/abc-def-ghi?resultId=123456789123456789
    ✓ GET - https://testing.website
```

### Reporters

Two reporters are supported out-of-the-box:

1. `stdout`
2. JUnit

To enable the JUnit report, pass the `--jUnitReport` (`-j` shorthand) in your command, specifying a filename for your JUnit XML report.

```bash
yarn datadog-ci synthetics run-tests -s 'tag:e2e-tests' --config global.config.json --jUnitReport e2e-test-junit
```

Reporters can hook themselves into the `MainReporter` of the command.

#### Available hooks

| Hook name        | Parameters                                                                               | Description                                                     |
| :--------------- | :--------------------------------------------------------------------------------------- | :-------------------------------------------------------------- |
| `log`            | `(log: string)`                                                                          | called for logging.                                             |
| `error`          | `(error: string)`                                                                        | called whenever an error occurs.                                |
| `initErrors`     | `(errors: string[])`                                                                     | called whenever an error occurs during the tests parsing phase. |
| `reportStart`    | `(timings: {startTime: number})`                                                         | called at the start of the report.                              |
| `resultEnd`      | `(result: Result, baseUrl: string)`                                                      | called for each result at the end of all results.               |
| `resultReceived` | `(result: Result)`                                                                       | called when a result is received.                               |
| `testTrigger`    | `(test: Test, testId: string, executionRule: ExecutionRule, config: UserConfigOverride)` | called when a test is triggered.                                |
| `testWait`       | `(test: Test)`                                                                           | called when a test is waiting to receive its results.           |
| `testsWait`      | `(tests: Test[])`                                                                        | called when all tests are waiting to receive their results.     |
| `runEnd`         | `(summary: Summary, baseUrl: string)`                                                    | called at the end of the run.                                   |
