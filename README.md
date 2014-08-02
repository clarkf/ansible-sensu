# Ansible Role: Sensu

Installs Sensu (either as a server or as a client) on servers.

At present, this role does __not__ support RabbitMQ TLS/SSL
communciation.  This is a planned addition.

## Supported Platforms

Currently this role only supports Debian-family hosts.  RedHat-family
hosts are a planned addition.

## Requirements

In general, Sensu has two dependencies: [RabbitMQ][rabbitmq] and
[Redis][redis].  This role doesn't assume that these are running on the
same machine.  Please have both installed and accessible and running.

## Configuration (as a Client)

To install Sensu as a client, simply override the variable
`sensu_rabbitmq`:

```yaml
sensu_rabbitmq:
  host: rmq01.mydomain.local
  port: 5672 # default RMQ port
  username: sensu_user
  password: sensu_password
```

If you'd like the client to identify itself as something other than its hostname
(not fully qualified), override `sensu_client_name`.  If you want custom
identifiers, it's recommended that you do this in your inventory file:

```ini
[webservers]
crimson.mydomain.local sensu_client_name=web01
```

### Advanced: Copying plugins

If you'd like this role to auto-magically copy over your plugins,
override the `sensu_plugins` variable:

```yaml
sensu_plugins:
  - files/sensu/plugins/check-procs.rb
  - files/sensu/plugins/cpu-metrics.rb
  # etc...
```

___Note___: Path handling here is handled by Ansible.  This typically
means that it will search for files first in this role's directory, then
in your present working directory.  Test this before a full-scale
deployment.

This files will end up in `/etc/sensu/plugins/`, but that's entirely
configurable.  You can override `sensu_plugins_directory` to instruct
the role to place them elsewhere.

## Configuration (as a Server)

Configuration as a server is almost the same as configuring as a client,
but you need to inform the role that you wish to configure and run the
API and Server daemons.

First, ensure that you have the correct services details defined:

```yaml
sensu_rabbitmq:
  host: rmq01.mydomain.local
  port: 5672 # default RMQ port
  username: sensu_user
  password: sensu_password

sensu_redis:
  host: rds01.mydomain.local
  port: 6379 # default Redis port

sensu_api:
  host: localhost
  port: 4567
  username: admin
  password: unicorns
```

Next, override `sensu_configs` to let the Role know that you'd like it
to also template the configuration files for the API and Redis (these
are not required on client nodes):

```yaml
sensu_configs:
  - client
  - rabbitmq
  - api
  - redis
```

This role will template those files, using the variables we defined
above.

Finally, we need to let the role which services to start.  Override
`sensu_services`:

```yaml
sensu_services:
  - sensu-server
  - sensu-api
  - sensu-client
```

And you'll have a minimal server installation. Congrats!

### Advanced: Copying extensions

Much like copying plugins to the host (in fact, you should do this on
server nodes too), this role will copy extensions over for you.  Simply
override `sensu_extensions`:

```
sensu_extensions:
  - files/sensu/extensions/flapjack.rb
  - files/sensu/extensions/graphite.rb
  # ...
```

This file will end up in `/etc/sensu/extensions/`. You can modify this
by overriding `sensu_extensions_directory`, but, be warned, Sensu only
automatically loads extensions from that path.

### Advanced: Generating checks

This role will automatically generate checks for you!  Check definitions
only need to be present on the server nodes, so take care to define the
following variables for those servers.

To add checks, override the `sensu_checks` variable:

```yaml
sensu_checks:
  - name: check_cron
    command: "{{ sensu_plugins_directory }}/check-procs.rb -d cron -C 1"
```

This is the simplest possible check.  You only need to specify a name
and a command, the rest will be taken care of.

Parameter | Req. | Type | Default | Description
----------|-----------|------|---------|------------
`name` | &#x2713; | string | n/a | The unique name of the check
`command` | &#x2713; | string | n/a | The command to be executed for the check
`subscribers` | &#x2717; | list | `{{sensu_default_subscribers}}` | An array of subscribers
`handlers` | &#x2717; | list | `{{sensu_default_handlers}}` | An array of handlers
`interval` | &#x2717; | int | `{{sensu_default_interval}}` | The interval (in seconds) at which this check should be performed
`type` | &#x2717; | string | n/a | The type of the check.  Add `metric` for metric checks.


So, an example of a more thorough check might be:

```yaml
- name: cpu_metrics
  command: "{{ sensu_plugins_directory }}/cpu-metrics.rb"
  type: metric
  subscribers:
    - webservers
    - databases
  handlers:
    - default
    - graphite
  interval: 15
```

## Example Playbook

__sensu-clients.yml__
```yaml
---
- hosts: all
  vars_files:
    - vars/sensu-clients.yml
  roles:
    - clarkf.sensu
```

__vars/sensu-clients.yml__
```yaml
---
sensu_rabbitmq:
  host: rmq01.mydomain.local
  port: 5672 # default RMQ port
  username: sensu_user
  password: sensu_password

sensu_plugins:
  - files/check-proc.rb
```

__sensu-servers.yml__
```yaml
---
- hosts: monitors
  vars_files:
    - vars/sensu-clients.yml
    - vars/sensu-servers.yml
  roles:
    - clarkf.sensu
```

__vars/sensu-servers.yml__
```yaml
---
sensu_redis:
  host: rds01.mydomain.local
  port: 6379 # default Redis port

sensu_api:
  host: localhost
  port: 4567
  username: admin
  password: unicorns

sensu_configs:
  - client
  - rabbitmq
  - api
  - redis

sensu_services:
  - sensu-server
  - sensu-api
  - sensu-client

sensu_extensions:
  - files/sensu/extensions/flapjack.rb
  - files/sensu/extensions/graphite.rb

sensu_checks:
  - name: check_cron
    command: "{{ sensu_plugins_directory }}/check-procs.rb -d cron -C 1"
  - name: check_ntp
    command: "{{ sensu_plugins_directory }}/check-procs.rb -d ntp -C 1"
```

## License

MIT.  See `LICENSE` for details.

## TODO

- Support for SSL/TLS RabbitMQ communication
- Support for RedHat-family operating systems (well... at least
  CentOS6).

## Contributing

See `contributing.md`.

[rabbitmq]: https://www.rabbitmq.com/
[redis]: http://redis.io/
