# WordPress

WordPress is a web content-management system.

## Resources

- [WordPress home page](https://wordpress.org)

	- [documentation](https://wordpress.org/documentation/)

- [DockerHub](https://hub.docker.com/_/wordpress)
- [GitHub](https://github.com/docker-library/wordpress)

## Overview

You need to perform two steps before WordPress can be launched:

1. [Install the service definition](#wpInst).
2. [Configure the environment](#wpConfig).

Note:

* Do **not** "up" your stack until you have completed step 2.

<a name="wpInst"></a>
## Install the service definition

Be in the correct directory:

``` console
$ cd ~/IOTstack
```

### option 1 - the IOTstack menu

1. Launch the menu

	``` console
	$ ./menu.sh
	```
	
2. Choose "Build Stack".
3. Place the cursor on "wordpress" and press <kbd>space</kbd> to select it.
4. Press <kbd>enter</kbd> to build the stack.
5. Place the cursor on "Exit" and press <kbd>enter</kbd>.

### option 2 - manual from IOTstack templates

When IOTstack is cloned from GitHub, the default for your local copy of the repository is to be on the "master" branch. Master-branch templates are left-shifted by two spaces with respect to how they need to appear in `docker-compose.yml`. The following `sed` command prepends two spaces to the start of each line:

``` console
$ sed -e "s/^/  /" ./.templates/wordpress/service.yml >>docker-compose.yml
```
						
Templates on the "old-menu" branch already have proper alignment, so `cat` can be used:

``` console
$ cat ./.templates/wordpress/service.yml >>docker-compose.yml
```

<a name="wpConfig"></a>
## Configure the environment

### check dependency

The password-generation steps in the [next section](#pwgen) assume `uuidgen` is available on your system. The following command installs `uuidgen` if it is not present already:

``` console
$ [ -z "$(which uuidgen)" ] && sudo apt update && sudo apt install -y uuid-runtime
```

<a name="pwgen"></a>
### generate passwords

WordPress relies on MariaDB, and MariaDB requires both a user password and a root password. You can generate the passwords like this:

``` console
$ echo "WORDPRESS_DB_PASSWORD=$(uuidgen)" >>~/IOTstack/.env
$ echo "WORDPRESS_ROOT_PASSWORD=$(uuidgen)" >>~/IOTstack/.env
```

Key points:

1. You will not need to know either of these passwords in order to use WordPress.

	> These passwords govern access to the WordPress database (the `wordpress_db` container). WordPress (the `wordpress` container) has a separate system of credentials. You set up an administrator account the first time you [login to WordPress](#wordPressGUI). 

2. You will not need to know either password in order to use the `mysql` command line interface to inspect the WordPress database. See [accessing the MariaDB command line interface](#mariaDBcli).
3. The WordPress database container does not expose any ports to the outside world. That means you can't use general-purpose MariaDB/MySQL GUI-based front-ends to reach the WordPress database.
4. Both passwords are applied when the MariaDB container is first initialised. Changing either password value in `.env` will break your installation.

<a name="setHostname"></a>
### set hostname

WordPress (running inside the container) needs to know the domain name of the host on which the container is running. You can satisfy the requirement like this:

``` console
$ echo "WORDPRESS_HOSTNAME=$HOSTNAME.local" >>~/IOTstack/.env
```

The above assumes the host is advertising a multicast domain name. This is a safe assumption for Raspberry Pis but may not necessarily be applicable in other situations. If your host is associated with a fully-qualified domain name (A record or CNAME), you can use that instead. For example:

``` console
$ echo "WORDPRESS_HOSTNAME=iot-hub.my.domain.com" >>~/IOTstack/.env
```

### checking your WordPress environment values

You can confirm that the passwords and hostname have been added to `.env` like this:

```
$ grep "^WORDPRESS" ~/IOTstack/.env
WORDPRESS_DB_PASSWORD=41dcbe76-9c39-4c7f-bd65-2f0421bccbeb
WORDPRESS_ROOT_PASSWORD=ee749d72-f1a5-4bc0-b182-21e8284f9fd2
WORDPRESS_HOSTNAME=raspberrypi.local
```

### alternative method

If you prefer to keep your environment values inline in your `docker-compose.yml` rather than in the `.env` file then you can achieve the same result by editing the service definitions as follows:

* `wordpress`:

	``` yaml
	  environment:
	    WORDPRESS_DB_PASSWORD: «yourUserPasswordHere»
	  hostname: «hostname».«domain»
	```

* `wordpress_db`:

	``` yaml
	  environment:
	    MYSQL_ROOT_PASSWORD: «yourRootPasswordHere»
	    MYSQL_PASSWORD: «yourUserPasswordHere»
	```

## Starting WordPress

``` console
$ cd ~/IOTstack
$ docker-compose up -d wordpress
```

This starts both WordPress and its database.

<a name="wordPressGUI"></a>
## Accessing the WordPress GUI

Use a URL in the following form, where `«host»` should be the value you chose at [set hostname](#setHostname).

```
http://«host»:8084
```

Examples:

* `http://raspberrypi.local:8084`
* `http://iot-hub.my.domain.com:8084`

You will be prompted to:

1. Set your language; and
2. Create your administrator account.

After that, you should refer to the [WordPress documentation](https://wordpress.org/documentation/).

<a name="aboutMariaDB"></a>
## About MariaDB

The MariaDB instance associated with WordPress is **private** to WordPress. It is included along with the WordPress service definition. You do **not** have to select MariaDB in the IOTstack menu.

> There is nothing stopping you from *also* selecting MariaDB in the IOTstack menu. Multiple instances of MariaDB will coexist quite happily but they are separate and distinct Relational Database Manager Systems (RDBMS).

<a name="mariaDBcli"></a>
### Accessing the MariaDB command line interface

If you need inspect or manipulate the WordPress database, begin by opening a shell into the WordPress MariaDB container:

```
$ docker exec -it wordpress_db bash
```

While you are in the shell, you can use the `MYSQL_ROOT_PASSWORD` environment variable to reference the root password. For example:

``` console
# mysql -p$MYSQL_ROOT_PASSWORD
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 169
Server version: 10.11.6-MariaDB-log Alpine Linux

Copyright (c) 2000, 2018, Oracle, MariaDB Corporation Ab and others.

Type 'help;' or '\h' for help. Type '\c' to clear the current input statement.

MariaDB [(none)]>
```

Note:

* There is no space between the `-p` and `$MYSQL_ROOT_PASSWORD`. If you insert a space, `mysql` will prompt you to enter the password interactively.

Once you have opened a session using `mysql`, you can execute MySQL commands. For example:

```
MariaDB [(none)]> show databases;
+--------------------+
| Database           |
+--------------------+
| information_schema |
| mysql              |
| performance_schema |
| sys                |
| wordpress          |
+--------------------+
5 rows in set (0.010 sec)
```

To exit `mysql`, either press <kbd>control</kbd>+<kbd>d</kbd> or use the `exit` command:

```
MariaDB [(none)]> exit
Bye

#
```

Similarly, <kbd>control</kbd>+<kbd>d</kbd> or `exit` will terminate the container's `bash` shell and return you to the host's command line.

## References to `nextcloud`

Both the `wordpress` and `wordpress_db` service definitions connect to the `nextcloud` **network**. 

> Please note the emphasis on "**network**".

The `nextcloud` network is an internal *private* network created by `docker-compose` to facilitate data-communications between a user-facing service (like WordPress) and an associated database back-end (like MariaDB).

The NextCloud container was the first to use the private-network strategy so the "nextcloud" name is an accident of history. In an ideal world, the network would be renamed to something which more accurately reflected its purpose, like "databases". Unfortunately, the IOTstack menu lacks the facilities needed to update *existing* deployments so the most likely result of any attempt at renaming would be to break existing stacks.

At runtime, the `nextcloud` network has the name `iotstack_nextcloud`, and exists alongside the `iotstack_default` network which is shared by other IOTstack containers.

The material point is that, even though WordPress has nothing to do with NextCloud, the references to the `nextcloud` network are are not mistakes. They are intentional.

## <a name="cleanSlate"></a>Getting a clean slate

If you start the WordPress container and *then* decide that you need to change its [environment variables](#wpConfig), you must first erase the container's persistent store:

``` console
$ cd ~/IOTstack
$ docker-compose down wordpress wordpress_db
$ sudo rm -rf ./volumes/wordpress
```

Notes:

* Both the `wordpress` and `wordpress_db` containers need to be taken down before the persistent store can be removed safely. 
* Be very careful with the `sudo rm` command. Double-check *before* pressing the <kbd>return</kbd> key!

Once the persistent store has been erased, you can change the [environment variables](#wpConfig).

When you are ready, start WordPress again:

``` console
$ cd ~/IOTstack
$ docker-compose up -d wordpress
```

Note:

* The `wordpress_db` container does not need to be brought up explicitly. It is started automatically as a by-product of starting `wordpress`.
