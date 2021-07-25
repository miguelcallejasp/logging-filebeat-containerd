# Logging with ContainerD

This is just to remember why I did things. I will post this in DZone because damn it's a freaking pain. 

### Architecture
The logs are being fetch from the following folder in the hosts:

```bash
/var/log/containers/*.log
```
The format of every file is:
```hcl
externaltransformer-eu-5fb985df68-hnb8c_green-test-io_externaltransformer-eu-c91682bb0c51ffeda3c4bee29d6bec9dbf75dbada5ced3d87e8dac128c87221f.log
```
The workflow goes:
- Filebeat (Daemon Set)
- Logstash (Deployment)
- Papertrail

## Filebeat
This is what the configuration does. This reads from all hosts on the path `/var/log/containers/*.log` 
and you can declare the files that will be excluded. This would work also as a prefilter for papertrail.

```yaml
    filebeat.inputs:
    - type: log
      enabled: true
      symlinks: true
      exclude_files: ['filebeat.*',
                      'logstash.*',
                      'azure.*',
                      'kube.*',
                      'ignite.*',
                      'influx.*',
                      'prometheus.*',
                      'rkubelog.*']
      paths:
        - /var/log/containers/*.log
```

Then we have the processors of those files.
```yaml
      - drop_fields:
          fields: ["host"]
          ignore_missing: true
      - dissect:
          tokenizer: "/var/log/containers/%{name}_%{host}_%{uuid}.log"
          field: "log.file.path"
          target_prefix: ""
          overwrite_keys: true
```
It first drops the `host` field because it contains the name of the node which
is irrelevant for the papertrail logs. 
Then the dissect part will get values from the filename which looks like this:

```shell
externaltransformer-eu-5fb985df68-hnb8c_green-test-io_externaltransformer-eu-c91682bb0c51ffeda3c4bee29d6bec9dbf75dbada5ced3d87e8dac128c87221f.log
```
and it will create the following fields: `name` (pod name), `host`(namespace), and `uuid` (pod UUID).
The target_prefix means that this fields will be created at the root of the message payload. And `log.file.path` is the 
field where the name of the filename is. 

```yaml
      - dissect:
          tokenizer: "%{header} F %{parsed}"
          field: "message"
          target_prefix: ""
          overwrite_keys: true
      - drop_fields:
          fields: ["message"]
          ignore_missing: true
```
This is because the logs in the kubernetes node have this format:

```shell
2021-07-08T20:34:07.979746229Z stderr F level=info ts=2021-07-08T20:34:07.978Z caller=node_exporter.go:112 collector=thermal_zone
```
So basically gets everything after the letter `F` and puts it in the `parsed` field.
Then it drops the message field because Logstash gets really confused. 

And finally, this renames the parsed field into message. 
```yaml
      - rename:
          fields:
            - from: "parsed"
              to: "message"
          ignore_missing: true
          fail_on_error: false
```

Some other small details are commented in the filebeats yaml file.

## Logstash
It receives the data from filebeat and merges 2 fields to format the data for Papertrail.
The connection is based on TCP outbound. Here's where the configuration for the destination needs to be done.

```shell
    input {
      beats {
          port => 5100
          host => "0.0.0.0"
          type => "log"
      }
    }

    filter {
      mutate {
        replace => { "message" => "%{name} %{message}" }
      }
    }

    output {
      tcp {
          codec => "line"
          host => "logs.papertrailapp.com"
          port => 59999
      }
    }
```

### Resources
Good resources here:
- https://dissect-tester.jorgelbg.me/