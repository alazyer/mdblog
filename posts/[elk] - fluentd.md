# Fluentd

## Components

The **input plugin** has responsibility for generating Fluentd event from data sources.

A **Filter** aims to behave like a rule to pass or reject an event. The *Filter* basically will accept or reject the *Event* based on **its *type* and rule defined**.

The new *Filter* definition added will be **a mandatory step before** to pass the control to the *Match* section.

Output plugin in **buffered mode** stores received events into buffers first and write out buffers to a destination by meeting **flush conditions**.

## Event

Fluentd event consists of tag, time and record.

- tag: Where an event comes from. For message routing
- time: When an event happens. Epoch time
- record: Actual log content. JSON object

## Life of a Fluentd Event



## Install Plugins

```bash
$ sudo /usr/sbin/td-agent-gem install fluent-plugin-s3
```

## Compoents

- You can use the `copy` [output plugin]() to **send the same event to multiple output destinations**.
- Use `"#{ENV['YOUR_ENV_VARIABLE']}"` to configure parameters dynamically with environment.
- To handle encoding errors:
  - Set Encoding correctly
    - `tail` input has encoding related parameters to change the log encoding
    - Use [`record_modifier`](https://github.com/repeatedly/fluent-plugin-record-modifier#char_encoding) filter to change the encoding
  - Use `yajl` instead of `json`

### Labels

```ini
<source>
  @type http
  bind 0.0.0.0
  port 8888
  @label @STAGING
</source>

<filter test.cycle>
  @type grep
  <exclude>
    key action
    pattern ^login$
  </exclude>
</filter>

<label @STAGING>
  <filter test.cycle>
    @type grep
    <exclude>
      key action
      pattern ^logout$
    </exclude>
  </filter>

  <match test.cycle>
    @type stdout
  </match>
</label>
```

当`curl -X POST -d 'json={"json":"message","action":"login"}' http://localhost:8888/test.cycle`时，influentd会将日志打印到console

当`curl -X POST -d 'json={"json":"message","action":"logout"}' http://localhost:8888/test.cycle`时，influentd不会讲日志打印到console