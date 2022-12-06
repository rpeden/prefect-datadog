# Sending Prefect Logs to Datadog

Prefect and Datadog are both great, and they're even greater when they work together. Sending Prefect logs to Datadog lets you use Datadog's log search, faceting, and notifications to keep track of how your tasks and flows are performing. 

Fortunately, integrating Prefect logs with Datadog only takes a few steps:
1. Create a custom Prefect `logging.yml` configuration that writes log entries to a file in JSON format.
2. Run Prefect with your custom logging configuration.
3. Install the Datadog agent.
4. Enable Datadog agent log collection.
5. Set up the Datadog Python plugin and tell it where to find your Prefect log file. 

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
 
You can add either of these handlers to the default `logging.yml`. 

To make things easy, this repository includes two Prefect logging config files - one for each `FileHandler`. Both include a new `datadog` entry in the `handlers` section. You can use either file as a starting point if you don't already use a customized `logging.yml`.

Let's examine each of them:

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
This handler caps log files at a specific size. Depending on your OS and machine setup, you may need to change `filename` to a location Prefect can write to. Note that you must provide the file's full path; using '~' as a shortcut to the current user's home directory won't work.

By default, this handler lets log files grow to 10MB. Adjust this to suit your needs by changing the `maxBytes` property. The `backupCount` property controls how many old log files the log handler keeps in addition to the current log file. 

Note that the handler uses the `json` formatter discussed above to ensure Prefect writes the log entries in a format Datadog can parse without additional configuration.

See [the Python docs](https://docs.python.org/3/library/logging.handlers.html#rotatingfilehandler) on `RotatingFileHandler` for additional configuration options. Any argument you can pass to the class constructor can be set in `logging.yml`.

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

This handler rotates log files by time interval. It's set to use one log file per day, and keep logs for 7 prior days. 

See the Python docs for more information about how you can configure `TimedRotatingFileHandler`. It offers many options for changing the logging interval and even the time of day when logs rollvoer to a new file. 

Now that you've added a Dataog log handler, it's time to use it. Look in the `loggers` section of the example files and see how the new handler has been added to several Prefect loggers:

```yaml
loggers:
    prefect:
        level: "${PREFECT_LOGGING_LEVEL}"

    prefect.extra:
        level: "${PREFECT_LOGGING_LEVEL}"
        handlers: [orion]

    prefect.flow_runs:
        level: NOTSET
        handlers: [orion, datadog]

    prefect.task_runs:
        level: NOTSET
        handlers: [orion, datadog]

    prefect.orion:
        level: "${PREFECT_LOGGING_SERVER_LEVEL}"

    prefect.client:
        level: "${PREFECT_LOGGING_LEVEL}"

    prefect.infrastructure:
        level: "${PREFECT_LOGGING_LEVEL}"
```

You may want to add your Datadog log handler to more or fewer loggers depending on your needs. See [the Prefect docs](https://docs.prefect.io/concepts/logs/) for more information on logging in Prefect.

## Running Prefect with custom logging configuration

Whether you use one of this repository's `.yml` files or create your own, the next step is getting Prefect to use your logging configuration. 

The easiest way is putting your logging configuration in a file named `logging.yml` in Prefect's home directory. The default Prefect home directory varies by operating system:
* `/home/<your username>/.prefect` on Linux
* `/Users/<your username>/.prefect` on MacOS
* `C:\Users\<your username>\.prefect` on Windows

Alternatively, you can keep your logging configuration elsewhere and run the following to tell Prefect where to look for logging configuration:
```bash
prefect config set PREFECT_LOGGING_SETTINGS_PATH=/path/to/your/prefect.log
```

## Install the Datadog agent
Instructions for installing the Datadog agent vary depending on your operating system. Fortunately, Datadog's docs guide you through the process. Start on [the Agent overview page](https://docs.datadoghq.com/agent/), select your platform, and follow the instructions to set up the agent.

## Enable Datadog agent log monitoring
By default, the Datadog agent does not collect logs. Fortunately, enabling it is easy. Start by finding your Datadog agent configuration directory. Its default location is OS-dependent:

* `/etc/datadog-agent` on Linux
* `~/.datadog-agent/` on MacOS
* `C:\ProgramData\Datadog\` on Windows

Navigate to this folder and open the `datadog.yaml` file. Search for `logs_enabled`, and you will find:
```yaml
    logs_enabled: false
```

Change the value so it reads:

```
    logs_enabled: true
```

Then, save the file and restart the Datadog agent to ensure your updated configuration works.

Alternatively, you can set the `DD_LOGS_ENABLED` environment variable to `true` and restart the Datadog agent. If you go this route, make sure you set the environment variable in a way that persists across server restarts, otherwise the Datadog agent will revert to its default of not collecting logs.

## Set up the Datadog Python plugin

The final step is enabling the Datadog agent's Python plugin. Open the configuration directory from the previous step, then open the `conf.d` subdirectory. Inside `conf.d`, create a subdirectory named `python.d`. Finally, inside `python.d`, create a file named `conf.yaml` with the following contents:

```yaml
init_config:

instances:

logs:

  - type: file
    path: "<path to your Prefect log>"
    service: "<A descriptive name of your Prefect service>"
    source: python
    sourcecategory: sourcecode
```

Update `path` with the full path to the log file used in your Prefect log configuration. Then, update service with a descriptive name. Depending on your needs, you might choose a generic name like *Prefect flows*, or something more specific like *Prefect Orion server* or *Prefect ML training agent*. Consider using the same service name in several locations if you run multiple agents or run your flows in containers and want all your Prefect log entries grouped together in Datadog. 

Next, restart the Datadog agent. Then, run a Prefect flow that generates log entries that you expect to show up given the logging settings you configured earlier. After a short delay, you'll see the log entries appear in Datadog. To view them, sign into Datadog and click on `Logs` in the menu.
