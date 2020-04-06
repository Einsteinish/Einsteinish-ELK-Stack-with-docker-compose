# Elasticsearch stack (ELK) with docker-compose

Run the latest version of the [Elastic stack][elk-stack] with Docker and Docker Compose.

Additional information is available from [Docker - ELK 7.6 : Elastic Stack with Docker Compose](https://www.bogotobogo.com/DevOps/Docker/Docker_ELK_7_6_Elastic_Stack_Docker_Compose.php) 

It gives us the ability to analyze any data set by using the searching/aggregation capabilities of Elasticsearch and
the visualization power of Kibana.

Based on the official Docker images from Elastic:

* [Elasticsearch](https://github.com/elastic/elasticsearch/tree/master/distribution/docker)
* [Logstash](https://github.com/elastic/logstash/tree/master/docker)
* [Kibana](https://github.com/elastic/kibana/tree/master/src/dev/build/tasks/os_packages/docker_generator)

## Contents

1. [Requirements](#requirements)
   * [Host setup](#host-setup)
2. [Usage](#usage)
   * [Version selection](#version-selection)
   * [Bringing up the stack](#bringing-up-the-stack)
   * [Cleanup](#cleanup)
   * [Initial setup](#initial-setup)
     * [Setting up user authentication](#setting-up-user-authentication)
     * [Injecting data](#injecting-data)
     * [Default Kibana index pattern creation](#default-kibana-index-pattern-creation)
3. [Configuration](#configuration)
   * [How to configure Elasticsearch](#how-to-configure-elasticsearch)
   * [How to configure Kibana](#how-to-configure-kibana)
   * [How to configure Logstash](#how-to-configure-logstash)
   * [How to enable paid features](#how-to-enable-paid-features)
   * [How to scale out the Elasticsearch cluster](#how-to-scale-out-the-elasticsearch-cluster)
4. [JVM tuning](#jvm-tuning)
   * [How to specify the amount of memory used by a service](#how-to-specify-the-amount-of-memory-used-by-a-service)
   * [How to enable a remote JMX connection to a service](#how-to-enable-a-remote-jmx-connection-to-a-service)
5. [Going further](#going-further)
   * [Plugins and integrations](#plugins-and-integrations)
   * [Swarm mode](#swarm-mode)
6. [Note](#note)

## Requirements

### Host setup

* [Docker Engine](https://docs.docker.com/install/) version **17.05** or newer
* [Docker Compose](https://docs.docker.com/compose/install/) version **1.20.0** or newer
* 1.5 GB of RAM

By default, the stack exposes the following ports:
* 5000: Logstash TCP input
* 9200: Elasticsearch HTTP
* 9300: Elasticsearch TCP transport
* 5601: Kibana

## Usage

### Version selection

This repository tries to stay aligned with the latest version of the Elastic stack. The `master` branch tracks the
current major version (7.x).

To use a different version of the core Elastic components, simply change the version number inside the `.env` file. If
we are upgrading an existing stack, please carefully read the note in the next section.


### Bringing up the stack

Clone this repository onto the Docker host that will run the stack, then start services locally using Docker Compose:

```console
$ docker-compose up
```

We can also run all services in the background (detached mode) by adding the `-d` flag to the above command.

> :warning: We must rebuild the stack images with `docker-compose build` whenever we switch branch or update the
> version of an already existing stack.

To start the stack for the very first time, please read the section below attentively.

### Cleanup

Elasticsearch data is persisted inside a volume by default.

In order to entirely shutdown the stack and remove all persisted data, use the following Docker Compose command:

```console
$ docker-compose down -v
```

## Initial setup

### Setting up user authentication

> :information_source: Refer to [How to enable paid features](#how-to-enable-paid-features) to enable authentication.

The stack is pre-configured with the following **privileged** bootstrap user:

* user: *elastic*
* password: *changeme*

Although all stack components work out-of-the-box with this user, we strongly recommend using the unprivileged [built-in
users][builtin-users] instead for increased security.

1. Initialize passwords for built-in users

```console
$ docker-compose exec -T elasticsearch bin/elasticsearch-setup-passwords auto --batch
```

Passwords for all 6 built-in users will be randomly generated. Take note of them.

2. Unset the bootstrap password (_optional_)

Remove the `ELASTIC_PASSWORD` environment variable from the `elasticsearch` service inside the Compose file
(`docker-compose.yml`). It is only used to initialize the keystore during the initial startup of Elasticsearch.

3. Replace usernames and passwords in configuration files

Use the `kibana` user inside the Kibana configuration file (`kibana/config/kibana.yml`) and the `logstash_system` user
inside the Logstash configuration file (`logstash/config/logstash.yml`) in place of the existing `elastic` user.

Replace the password for the `elastic` user inside the Logstash pipeline file (`logstash/pipeline/logstash.conf`).

> :information_source: Do not use the `logstash_system` user inside the Logstash *pipeline* file, it does not have
> sufficient permissions to create indices. Follow the instructions at [Configuring Security in Logstash][ls-security]
> to create a user with suitable roles.

See also the [Configuration](#configuration) section below.

4. Restart Kibana and Logstash to apply changes

```console
$ docker-compose restart kibana logstash
```

> :information_source: Learn more about the security of the Elastic stack at [Tutorial: Getting started with
> security][sec-tutorial].

### Injecting data

Give Kibana about a minute to initialize, then access the Kibana web UI by hitting
[http://localhost:5601](http://localhost:5601) with a web browser and use the following default credentials to log in:

* user: *elastic*
* password: *\<generated elastic password>*

Now that the stack is running, we can go ahead and inject some log entries. The shipped Logstash configuration allows 
us to send content via TCP:


```console
# Using BSD netcat (Debian, Ubuntu, MacOS system, ...)
$ cat /path/to/logfile.log | nc -q0 localhost 5000
```

```console
# Using GNU netcat (CentOS, Fedora, MacOS Homebrew, ...)
$ cat /path/to/logfile.log | nc -c localhost 5000
```

We can also load the sample data provided by our Kibana installation.

### Default Kibana index pattern creation

When Kibana launches for the first time, it is not configured with any index pattern.

#### Via the Kibana web UI

> :information_source: We need to inject data into Logstash before being able to configure a Logstash index pattern via
the Kibana web UI.

Navigate to the _Discover_ view of Kibana from the left sidebar. We will be prompted to create an index pattern. Enter
`logstash-*` to match Logstash indices then, on the next page, select `@timestamp` as the time filter field. Finally,
click _Create index pattern_ and return to the _Discover_ view to inspect our log entries.

Refer to [Connect Kibana with Elasticsearch][connect-kibana] and [Creating an index pattern][index-pattern] for detailed
instructions about the index pattern configuration.

#### On the command line

Create an index pattern via the Kibana API:

```console
$ curl -XPOST -D- 'http://localhost:5601/api/saved_objects/index-pattern' \
    -H 'Content-Type: application/json' \
    -H 'kbn-version: 7.6.2' \
    -u elastic:<generated elastic password> \
    -d '{"attributes":{"title":"logstash-*","timeFieldName":"@timestamp"}}'
```

The created pattern will automatically be marked as the default index pattern as soon as the Kibana UI is opened for the first time.

## Configuration

> :information_source: Configuration is not dynamically reloaded, we will need to restart individual components after
any configuration change.

### How to configure Elasticsearch

The Elasticsearch configuration is stored in [`elasticsearch/config/elasticsearch.yml`][config-es].

We can also specify the options we want to override by setting environment variables inside the Compose file:

```yml
elasticsearch:

  environment:
    network.host: _non_loopback_
    cluster.name: my-cluster
```

Please refer to the following documentation page for more details about how to configure Elasticsearch inside Docker
containers: [Install Elasticsearch with Docker][es-docker].

### How to configure Kibana

The Kibana default configuration is stored in [`kibana/config/kibana.yml`][config-kbn].

It is also possible to map the entire `config` directory instead of a single file.

Please refer to the following documentation page for more details about how to configure Kibana inside Docker
containers: [Running Kibana on Docker][kbn-docker].

### How to configure Logstash

The Logstash configuration is stored in [`logstash/config/logstash.yml`][config-ls].

It is also possible to map the entire `config` directory instead of a single file, however we must be aware that
Logstash will be expecting a [`log4j2.properties`][log4j-props] file for its own logging.

Please refer to the following documentation page for more details about how to configure Logstash inside Docker
containers: [Configuring Logstash for Docker][ls-docker].

### How to enable paid features

Switch the value of Elasticsearch's `xpack.license.self_generated.type` option from `basic` to `trial`(see [License
settings][trial-license]).

### How to scale out the Elasticsearch cluster

Follow the instructions from the Wiki: [Scaling out Elasticsearch](https://github.com/deviantony/docker-elk/wiki/Elasticsearch-cluster)

## Extensibility

### How to add plugins

To add plugins to any ELK component we have to:

1. Add a `RUN` statement to the corresponding `Dockerfile` (eg. `RUN logstash-plugin install logstash-filter-json`)
2. Add the associated plugin code configuration to the service configuration (eg. Logstash input/output)
3. Rebuild the images using the `docker-compose build` command

## JVM tuning

### How to specify the amount of memory used by a service

By default, both Elasticsearch and Logstash start with [1/4 of the total host
memory](https://docs.oracle.com/javase/8/docs/technotes/guides/vm/gctuning/parallel.html#default_heap_size) allocated to
the JVM Heap Size.

The startup scripts for Elasticsearch and Logstash can append extra JVM options from the value of an environment
variable, allowing the user to adjust the amount of memory that can be used by each component:

| Service       | Environment variable |
|---------------|----------------------|
| Elasticsearch | ES_JAVA_OPTS         |
| Logstash      | LS_JAVA_OPTS         |

To accomodate environments where memory is scarce (Docker for Mac has only 2 GB available by default), the Heap Size
allocation is capped by default to 256MB per service in the `docker-compose.yml` file. If we want to override the
default JVM configuration, edit the matching environment variable(s) in the `docker-compose.yml` file.

For example, to increase the maximum JVM Heap Size for Logstash:

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Xmx1g -Xms1g
```

### How to enable a remote JMX connection to a service

As for the Java Heap memory (see above), we can specify JVM options to enable JMX and map the JMX port on the Docker
host.

Update the `{ES,LS}_JAVA_OPTS` environment variable with the following content (I've mapped the JMX service on the port
18080, we can change that). Do not forget to update the `-Djava.rmi.server.hostname` option with the IP address of our
Docker host (replace **DOCKER_HOST_IP**):

```yml
logstash:

  environment:
    LS_JAVA_OPTS: -Dcom.sun.management.jmxremote -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Dcom.sun.management.jmxremote.port=18080 -Dcom.sun.management.jmxremote.rmi.port=18080 -Djava.rmi.server.hostname=DOCKER_HOST_IP -Dcom.sun.management.jmxremote.local.only=false
```

## Going further

### Plugins and integrations

See the following Wiki pages:

* [External applications](https://github.com/deviantony/docker-elk/wiki/External-applications)
* [Popular integrations](https://github.com/deviantony/docker-elk/wiki/Popular-integrations)

### Swarm mode

Experimental support for Docker [Swarm mode][swarm-mode] is provided in the form of a `docker-stack.yml` file, which can
be deployed in an existing Swarm cluster using the following command:

```console
$ docker stack deploy -c docker-stack.yml elk
```

If all components get deployed without any error, the following command will show 3 running services:

```console
$ docker stack services elk
```

> :information_source: To scale Elasticsearch in Swarm mode, configure *zen* to use the DNS name `tasks.elasticsearch`
instead of `elasticsearch`.


[elk-stack]: https://www.elastic.co/elk-stack
[stack-features]: https://www.elastic.co/products/stack
[paid-features]: https://www.elastic.co/subscriptions
[trial-license]: https://www.elastic.co/guide/en/elasticsearch/reference/current/license-settings.html

[linux-postinstall]: https://docs.docker.com/install/linux/linux-postinstall/

[booststap-checks]: https://www.elastic.co/guide/en/elasticsearch/reference/current/bootstrap-checks.html
[es-sys-config]: https://www.elastic.co/guide/en/elasticsearch/reference/current/system-config.html

[win-shareddrives]: https://docs.docker.com/docker-for-windows/#shared-drives
[mac-mounts]: https://docs.docker.com/docker-for-mac/osxfs/

[builtin-users]: https://www.elastic.co/guide/en/elasticsearch/reference/current/built-in-users.html
[ls-security]: https://www.elastic.co/guide/en/logstash/current/ls-security.html
[sec-tutorial]: https://www.elastic.co/guide/en/elasticsearch/reference/current/security-getting-started.html

[connect-kibana]: https://www.elastic.co/guide/en/kibana/current/connect-to-elasticsearch.html
[index-pattern]: https://www.elastic.co/guide/en/kibana/current/index-patterns.html

[config-es]: ./elasticsearch/config/elasticsearch.yml
[config-kbn]: ./kibana/config/kibana.yml
[config-ls]: ./logstash/config/logstash.yml

[es-docker]: https://www.elastic.co/guide/en/elasticsearch/reference/current/docker.html
[kbn-docker]: https://www.elastic.co/guide/en/kibana/current/docker.html
[ls-docker]: https://www.elastic.co/guide/en/logstash/current/docker-config.html

[log4j-props]: https://github.com/elastic/logstash/tree/7.6/docker/data/logstash/config
[esuser]: https://github.com/elastic/elasticsearch/blob/7.6/distribution/docker/src/docker/Dockerfile#L23-L24

[upgrade]: https://www.elastic.co/guide/en/elasticsearch/reference/current/setup-upgrade.html

[swarm-mode]: https://docs.docker.com/engine/swarm/

## Note
This repository is a forked/trimmed version of [docker-elk](https://github.com/deviantony/docker-elk)
