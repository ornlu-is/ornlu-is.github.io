# Hassle-free table creation on start up for Clickhouse Docker containers


This is a neat little trick I learned recently. I was in need of creating a Clickhouse container to attach to another application that would write to it. And I found myself thinking: it would be really convenient if there were a way to create a Clickhouse container that already comes with a set of tables created. Most of the solutions I found only were severely convoluted and relied on scripts to make this happen, so I thought that there must surely be a better way. As it turns out, there is, and this post is precisely about that.

## The Docker image

The first step is to specify our Clickhouse `Dockerfile`:

```Dockerfile
FROM yandex/clickhouse-server

ADD ./docker-entrypoint-initdb.d/ /docker-entrypoint-initdb.d 
```

Basically, all we do is take the base Clickhouse image and copy a local directory called `docker-entrypoint-initdb.d` into a folder of the same name inside the container. This is a special folder that Clickhouse will peer inside when being started as a Docker container. It will look for SQL scripts inside this directory and execute them on start up. Which means that, if we want a Clickhouse Docker image that already has a few existing tables, all we have to do is specify a few SQL scripts and write an exceedingly simple `Dockerfile`!

## The SQL script

This brings us to the second step: writing a basic SQL script so we can see this in action. Inside the `docker-entrypoint-initdb.d` directory, let's write the following script:

```sql
CREATE TABLE IF NOT EXISTS random_table
(
    `field1` Float,
    `field2` Float,
    `field3` Float
)
ENGINE = Memory()
```

We will name our script `1_create_random_table.sql` so that we know what it does and we can be sure that it is the first script to run (assuming other scripts are also prefixed by numbers). Another important detail is that we added added the `IF NOT EXISTS` modifier to our `CREATE TABLE` statement. This was no accident. To understand its purpose, consider the following scenario. You build and start your container and you see your tables have been written to Clickhouse, and, to celebrate this success, you stop the container a go grab a quick beer. While drinking the aforementioned beer, you brag to someone about your most recent accomplishment, but they do not believe you. With a fiery smirk, you prompt that someone to follow you to your computer. And there you go, half empty beer in hand, confidently walking to your computer. You sit down, type the command to start the container and... your container exits with a failure code immediately after starting. Your beer is suddenly warm, your confidence is gone, and the embarassment you feel from this debacle will keep you awake at night and eventually be the main cause of your divorce.

Now, the lesson here is simple: without the `IF NOT EXISTS` modifier, when you run the container a second time without removing the leftover container from the first time you ran it, it will attempt to create the table again, and fail because it already exists, and then you'll end up going through a ravaging divorce. Nobody wants that, so use this modifier.

## Inspecting the container

Considering our SQL script is now divorce-proof, all that is left is for us is to test this out. In the same directory as our `Dockerfile`, simply run:
```plaintext
docker build . -t 'ch-sql'
```
where `ch-sql` is the tag I decided to give this image, but you can call it whatever you'd like. Now, when Docker finishes building the image, we can check its existence by inspecting the output of the `docker images` command:

```plaintext
REPOSITORY          TAG       IMAGE ID       CREATED          SIZE
ch-sql              latest    c3ec2b44d1e8   46 seconds ago   826MB
```

Great, since our image is there, we are all set to create the container using `docker run ch-sql`, which outputs the following:
```plaintext
Processing configuration file '/etc/clickhouse-server/config.xml'.
Merging configuration file '/etc/clickhouse-server/config.d/docker_related_config.xml'.
Logging trace to /var/log/clickhouse-server/clickhouse-server.log
Logging errors to /var/log/clickhouse-server/clickhouse-server.err.log
Processing configuration file '/etc/clickhouse-server/config.xml'.
Merging configuration file '/etc/clickhouse-server/config.d/docker_related_config.xml'.
Saved preprocessed configuration to '/var/lib/clickhouse/preprocessed_configs/config.xml'.
Processing configuration file '/etc/clickhouse-server/users.xml'.
Saved preprocessed configuration to '/var/lib/clickhouse/preprocessed_configs/users.xml'.

/entrypoint.sh: running /docker-entrypoint-initdb.d/1_create_random_table.sql


Processing configuration file '/etc/clickhouse-server/config.xml'.
Merging configuration file '/etc/clickhouse-server/config.d/docker_related_config.xml'.
Logging trace to /var/log/clickhouse-server/clickhouse-server.log
Logging errors to /var/log/clickhouse-server/clickhouse-server.err.log
Processing configuration file '/etc/clickhouse-server/config.xml'.
Merging configuration file '/etc/clickhouse-server/config.d/docker_related_config.xml'.
Saved preprocessed configuration to '/var/lib/clickhouse/preprocessed_configs/config.xml'.
Processing configuration file '/etc/clickhouse-server/users.xml'.
Saved preprocessed configuration to '/var/lib/clickhouse/preprocessed_configs/users.xml'.
```

I did not add the empty lines that separate the our script's execution, Clickhouse did that itself, which is pretty convenient. All that is left is to `exec` into the container and examine if our table was actually created. Grab the container ID from the output of `docker ps`, and open an interactive terminal session inside the container with:
```plaintext
docker exec -it <CONTAINER-ID> bash 
```
If you've managed to get here without any mistakes, you should now be inside the container image we built. Starting the Clickhouse client is very simple, all you have to do is type `clickhouse-client` and press Enter, and you'll be presented with a Clickhouse prompt:

```plaintext
ClickHouse client version 22.1.3.7 (official build).
Connecting to localhost:9000 as user default.
Connected to ClickHouse server version 22.1.3 revision 54455.

CONTAINER_ID :) 
```

To check if our table exists, simply write `SHOW TABLES FROM default`, where `default` is the default database where we've created our table, well, by default. This will give us the following:

```sql
SHOW TABLES FROM default

Query id: 8052df88-1463-487a-a31d-9e8c504181c3

┌─name─────────┐
│ random_table │
└──────────────┘

1 rows in set. Elapsed: 0.005 sec.
```

So now you're all set to execute start up SQL scripts on a Clickhouse Docker container, hopefully this will end up saving your marriage! 

{{< admonition type=tip title="Link to the code" open=true >}}
https://github.com/ornlu-is/clickhouse_docker_start_up_scripts
{{< /admonition >}}

