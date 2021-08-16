![Flight Deck](https://raw.githubusercontent.com/ten7/flight-deck/master/flightdeck-logo.png)

# Flight Deck DB

Flight Deck DB is a minimalist MySQL/mariaDB container for Drupal sites on Kubernetes and Docker. You can use it both for local development and production.

Features:
* ConfigMap-friendly YAML configuration
* Supports multiple users and databases

## Tags and versions

There are several tags available for this container:

| Tags | mariaDB version | MySQL equiv. |
| ---- | --------------- | ------------ |
| latest | 10.3 | 5.7 |
| x.y.z | 10.3 | 5.7 |

## Configuration

Instead of a large number of environment variables, this container relies on a file to perform all runtime configuration, `flightdeck-db.yml`.

You can provide this file in one of three ways to the container:

* Mount the configuration file at path `/config/web/flightdeck-db.yml` inside the container using a bind mount, configmap, or secret.
* Mount the config file anywhere in the container, and set the `FLIGHTDECK_CONFIG_FILE` environment variable to the path of the file.
* Encode the contents of `flightdeck-varnish.yml` as base64 and assign the result to the `FLIGHTDECK_CONFIG` environment variable.

### Basic settings

You can set the root password, as well as key buffer and cache sizes using the following variables:

```yaml
---
mysql_root_password: "root"
mysql_key_buffer_size: "256M"
mysql_max_allowed_packet: "64M"
mysql_table_open_cache: "256"
mysql_query_cache_size: "0"
```

Where:
* **mysql_root_password** is the MySQL root password to use. Required unless `mysql_root_password_file` is defined.
* **mysql_key_buffer_size** is the key buffer size to use. Optional, defaults to `256M`
* **mysql_max_allowed_packet** is the maximum allowed packet size. Optional, defaults to `64M`.
* **mysql_table_open_cache** is the table open cache count. Optional, defaults to `256`.
* **mysql_query_cache_size** is the query cache size to retain. Optional, defaults to `0` as the query cache can be counterproductive for Drupal sites.

### Defining databases

This container supports multiple databases by defining the `mysql_databases` list:

```yaml
---
mysql_databases:
  - name: "drupal"
    state: present
    encoding: "latin1"
    collation: "latin1_general_ci"
```

Each item has the following variables:

* **name** is the name of the database name. Required.
* **state** is if the database is `present` or `absent`. Specify `absent` to delete an existing database. Optional, defaults to `present`.
* **encoding** is the type encoding scheme of the database. Optional, defaults to `utf8`.
* **collation** is the database collation to use. Optional, defaults to `utf8_general_ci`

### Defining users

This container also supports multiple users by defining the `mysql_users` list:

```yaml
---
mysql_users:
  - name: "drupal"
    state: present
    host: "%"
    password: "drupal"
    priv: "drupal.*:ALL"
```

Each item has the following variables:

* **name** is the username to create. Required.
* **state** is if the user is `present` or `absent`. Specify `absent` to delete existing users. Optional, defaults to `present`.
* **host** is the host from which the user may use their account. While optional, it is best practice to set this in nearly all cases.
* **password** is the user's password. Optional. Ignored if `passwordFile` is defined.
* **priv** is a list of database privileges in the form of `database.table:privilege`. Multiple privileges can be specified with a comma (`,`), multiple databases are separated with a `/`. Optional, but highly recommended.

### Using separate files for credentials

Sometimes, you may wish to keep certain credentials in separate files from the rest of the configuration. One such case is if you want to keep `flight-deck-db.yml` in a Kubernetes configmap, but keep the database passwords in secrets instead.

```yaml
mysql_root_password_file: /config/mysql-root-user/password.txt
mysql_users:
  - name: "drupal"
    state: present
    host: "%"
    passwordFile: "/config/mysql-drupal-user/password.txt"
    priv: "drupal.*:ALL"
```

Where:

* **mysql_root_password_file** is the full path to a file containing the password for the `root` account.
* **passwordFile** is the full path to a file containing the user's password.

All paths are to the file *inside* the container.

## Deployment on Kubernetes

Use the [`ten7.flightdeck_cluster`](https://galaxy.ansible.com/ten7/flightdeck_cluster) role on Ansible Galaxy to deploy DB as a statefulset:

```yaml
flightdeck_cluster:
  namespace: "database"
  secrets:
    - name: "flight-deck-db"
      files:
        - name: "flight-deck-db.yml"
          content: |
            mysql_root_password: "root"
            mysql_databases:
              - name: "drupal"
                encoding: "latin1"
                collation: "latin1_general_ci"
            mysql_users:
              - name: "drupal"
                host: "%"
                password: "drupal"
                priv: "drupal.*:ALL"
            mysql_key_buffer_size: "256M"
            mysql_max_allowed_packet: "64M"
            mysql_table_open_cache: "256"
            mysql_query_cache_size: "0"
  mysql:
    size: "10Gi"
    secrets:
      - name: "flight-deck-db"
        path: "/config/mysql"
```

## Using with Docker Compose

Create the `flight-deck-db.yml` file relative to your `docker-compose.yml`. Define the `db` service mounting the file as a volume:

```yaml
version: '3'
services:
  db:
    image: ten7/flight-deck-db:10
    ports:
      - 3306:3306
    volumes:
      - /var/lib/mysql
      - ./db-backups:/tmp/db-backups:cached
      - ./flight-deck-db.yml:/config/mysql/flight-deck-db.yml
```

## Part of Flight Deck

This container is part of the [Flight Deck library](https://github.com/ten7/flight-deck) of containers for Drupal local development and production workloads on Docker, Swarm, and Kubernetes.

Flight Deck is used and supported by [TEN7](https://ten7.com/).

## Debugging

If you need to get verbose output from the entrypoint, set `flightdeck_debug` to `true` or `yes` in the config file.

```yaml
---
flightdeck_debug: yes
```

This container uses [Ansible](https://www.ansible.com/) to perform start-up tasks. To get even more verbose output from the start up scripts, set the `ANSIBLE_VERBOSITY` environment variable to `4`.

If the container will not start due to a failure of the entrypoint, set the `FLIGHTDECK_SKIP_ENTRYPOINT` environment variable to `true` or `1`, then restart the container.

### Skipping authorization

If you've been locked out of the database due to a permissions problem, you can override the default database startup command to start without authentication and networking.

In Kubernetes:

Edit the database Deployment or StatefulSet to override the `flight-deck-db` container's default `command`. Optionally, add an `env` section to set debug flags as described above.

```yaml
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mysql
spec:
  template:
    spec:
      containers:
      - image: ten7/flight-deck-db:develop
        command:
          - mysqld_safe
          - --skip-grant-tables
          - --skip-networking
        env:
          - name: FLIGHTDECK_SKIP_ENTRYPOINT
            value: "1"
          - name: ANSIBLE_VERBOSITY
            value: "4"
```

In Docker Compose:

Edit your `docker-compose.yml` file to override the `command` the `flight-deck-db` container. Optionally, add an `environment` section to set debug flags as described above.

```yaml
version: '3'
services:
  db:
    image: ten7/flight-deck-db:10
    command: ["mysqld_safe", "--skip-grant-tables", "--skip-networking"]
    environment:
      FLIGHTDECK_SKIP_ENTRYPOINT: 1
      ANSIBLE_VERBOSITY: 4
```

## License

Flight Deck is licensed under GPLv3. See [LICENSE](https://raw.githubusercontent.com/ten7/flightdeck-db-10.3/master/LICENSE) for the complete language.
