# Sending Prefect Logs to Datadog

Prefect and Datadog are both great, and they're even greater when they work together. Sending Prefect logs to Datadog lets you use Datadog's log search, faceting, and notifications to keep track of how your tasks and flows are performing. 

Fortunately, integrating Prefect logs with Datadog only quires a few easy steps:
1. Create a custom Prefect `logging.yml` configuration that writes log entries to a file in JSON format.
2. Run Prefect with your custom logging configuration.
3. Install the Datadog agent.
4. Set up the Datadog Python plugin and tell it where to find your Prefect log file. 

Let's dive in and cover each step in detail. 
  

## Create custom Prefect logging configuration
The Datadog agent reads log entries from files, and JSON log entries are the easiest way to get log data into Datadog. By default, Prefect doesn't output JSON logs, nor does it save logs to a file. Fortunately, adding both is easy. 

A quick look at Prefect's [default logging configuration](https://github.com/PrefectHQ/prefect/blob/main/src/prefect/logging/logging.yml) shows there's already a JSON formatter available:

```yaml
json:
    class: prefect.logging.formatters.JsonFormatter
    format: "default"
```

All we need to do is use it when setting up a file logger. You'll want to set up rotating logs that ensure you don't accidentally fill your disk with log entries. Fortunately, Python's `logging.handlers` module includes two built-in log handlers that handle log file rotation: 
  
 * `RotatingFileHandler` — rotates logs based on log file size.
 * `TimedRotatingFileHandler` — rotates logs by time period.
 
Either of these handlers to the default `logging.yml` configuration. To make things easy, this repository includes two Prefect logging config files - one for each `FileHandler`. Both include a new `datadog` entry in the `handlers` section. Let's take look at each of them:

**size-rotating-logging.yml**
```yaml
datadog:
    level: 0
    class: logging.handlers.RotatingFileHandler
    filename: /var/log/prefect.log
    # maximum 10MB per log file
    maxBytes: 10485760
    # keep the last 5 log files in addition to the current log file
    backupCount: 5
    formatter: json
```

**time-rotating-logging.yml**
```yaml
datadog:
    level: 0
    class: logging.handlers.TimedRotatingFileHandler
    filename: /var/log/prefect.log
    when: 'D'
    interval: 1
    backupCount: 7
    formatter: json
```
