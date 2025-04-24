# Logstash Write-up

## Project Setup & Learning Experience
To finish this exercise, I relied heavily on Logstash documentation and GitHub repository (for Grok patterns)
* https://www.elastic.co/docs/reference/logstash
* https://github.com/logstash-plugins/logstash-patterns-core/tree/main/patterns/ecs-v1

## Learning Elastic Stack Basics
I started by watching several online tutorials to build foundational knowledge of Logstash and Elasticsearch, covering key concepts such as data pipelines, plugins (input, filter, output), and configuration structure. This gave me a clear understanding of how Logstash fits into the broader Elastic Stack ecosystem.
## Environment Setup
I provisioned a cloud-based Ubuntu server and created a dedicated user (logstash) with sudo access to isolate and manage the deployment. Logstash was installed using the official APT repository as outlined in Elastic's documentation:
* https://www.elastic.co/docs/reference/logstash/installing-logstash#_apt

Since on Ubuntu, logstash is installed as a systemd service, I enabled and started the service using:
```sh
sudo systemctl enable logstash
sudo systemctl start logstash
```

I also added the path to the Logstash binaries to my `$PATH` so I could run the `logstash` command when required (such as for testing if the configuration was `OK`) using the following command:
```sh
export PATH=$PATH:/usr/share/logstash/bin
```

I added this to the .bashrc file for the logstash user so that logstash command is used on every subsequent login.

For the configuration, I created my configuration file in the `/etc/logstash/conf.d/` directory. Any number of configuration files can be created here, and they are executed as pipelines by Logstash. I created mine with the name `logstash.conf`

I tested my configuration file iteratively at each step using the following command:
```sh
logstash -f /etc/logstash/conf.d/logstash.conf --path.settings /etc/logstash --config.test_and_exit
```

This command points out obvious syntax errors with the configuration, and otherwise returns `OK` if the syntax looks good.

That being said, when I started learning Logstash, I used the toy examples in the documentation to become familiar with how logstash works and how a pipeline is constructed using a basic input and output filter (standard input `stdin` to standard output `stdout`): https://www.elastic.co/docs/reference/logstash/first-event

Then, I saved the sample log shared in the exercise in a file called `test.log` in the logstash home directory: 
`/home/logstash/test.log`

Then, I switched the input in my configuration file from stdin to file plugin and specified the path to the log file.

This is where I ran into one of the first challenges - I noticed that the log file disappeared after the pipeline ran. After reading through the documentation further, I realized that this was due to the default behavior of the file plugin. To fix this, I modified the [`file_completed_action`](https://www.elastic.co/docs/reference/logstash/plugins/plugins-inputs-file#plugins-inputs-file-file_completed_action) as described here: https://www.elastic.co/docs/reference/logstash/plugins/plugins-inputs-file#plugins-inputs-file-file_completed_action

Once this was addressed, I similarly switched out the stdout output plugin and replaced it with the file output plugin. I tried to use both the json and json_lines codecs since the desired output is in JSON format. Both gave JSON output in the file. At this point the output was quite simply the original sample log itself.

After that, the important bit that was left was to extract information from the logs and also transform it as per the exercise. The hint was in the exercise, and I went through the filter plugins documentation.

After reading through the description of various filter plugins, I realized that grok filter plugin was a suitable plugin for the task, as it could use pattern matching to match the format of logs. It was also possible to assign user-friendly names to the matched patterns, and I was able to use those to get the correct JSON key names for 3 out of 4 fields in the exercise.

To match the patterns, I went through the Github repository where several Grok patterns have already been made available. Grok patterns are quite simply wrappers around the most common regular expressions. The task was the figure out the most appropriate pattern for each field in the log message.
 
For the severity field, I first extracted the raw severity level using Grok and gave it the name `severity_level`. Then I used the translate filter plugin to map the value `1` to `High` using `severity_level` as the source field and severity as the desired target field.

# Final Configuration

My final config is below.

```
input { 
    file {
        path => "/home/logstash/test.log"
        mode => "read"
        sincedb_path => "/dev/null"
        file_completed_action => "log"
        file_completed_log_path => "/home/logstash/out_completed.txt"
    }
}

filter {
    grok {
        match => {
            "message" => "^<%{NUMBER:line_num}>%{NUMBER:char_num} %{TIMESTAMP_ISO8601:timestamp} %{WORD:src} %{WORD:program} %{NUMBER:pid} - - alertname=\"%{DATA:description}\" computername=\"%{DATA:hostname}\" computerip=\"%{IP:source_ip}\" severity=\"%{NUMBER:severity_level}\""
        }
    }

    translate {
        field => "severity_level"
        destination => "severity"
        dictionary => {
            "0" => "Informational"
            "1" => "High"
            "2" => "Critical"
        }
    }
}

output {
    stdout {
        codec => "json_lines"
    }
    file {
        codec => "json"
        path => "/home/logstash/out.json"
    }
    file {
        codec => "json_lines"
        path => "/home/logstash/out_lines.json"
    }
}
```
