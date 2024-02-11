+++
authors = ["Agustín Ramiro Díaz"]
title = "Understanding Linux file permissions in Docker containers"
description = ""
date = 2024-02-10
[taxonomies]
tags = ["Docker", "Containers", "Linux"]
[extra]
toc = true
[extra.comments]
id = ""
+++

# Motivation

I recently wanted to work in the [surrealdb](https://github.com/surrealdb/surrealdb) codebase, and by looking at its issues I've found [one related to file permissions in Docker](https://github.com/surrealdb/surrealdb/issues/3468). I remember having some vague knowledge about issues with file permissions before, and I thought it would be a good time to clear my doubts.

# Understanding file permissions in Linux

I won't go into too much detail because there are [great resources](https://linuxhandbook.com/linux-file-permissions/) that explain this better than me. Heres's a brief summary:

In Linux, file permissions are managed by the kernel. Each file has an owner, a group, and a set of permissions. The owner is the user who created the file, and the group is a set of users who have the same permissions over the file. The permissions are divided into three categories: read, write, and execute. Each category has three possible values: allowed, denied, and not set. The permissions are represented by a 9-character string, where the first three characters represent the owner's permissions, the next three characters represent the group's permissions, and the last three characters represent the permissions for everyone else.

This permissions are usually handled with numbers or leters when running commands like `chmod`. The numbers are octal, and the letters are `r` for read, `w` for write, and `x` for execute. The numbers are calculated by adding the values of the permissions: 4 for read, 2 for write, and 1 for execute. For example, if we want to give read and write permissions to the owner, and read permissions to the group and everyone else, we would run `chmod 644 file.txt`. This would set the permissions to `-rw-r--r--`. We can also use the letters to set the permissions, for example, `chmod u+x file.txt` would add the execute permission to the owner.

# Understanding file permissions in Docker containers

Docker containers are isolated environments, but they still run on top of the host's kernel. This means that the file permissions are managed by the host's kernel, and the container's kernel is not involved in this process. This is important to understand because it means that the file permissions inside the container are the same as the file permissions on the host. This can lead to issues for example when the container and the host have different users and groups.

By default, the container user is set to `root` with uid and gid 0. This means that if we create a file inside the container, `root` will be the owner of the file. This can lead to issues when we try to access the file from the host, because the host's user might not have the same uid as the container's `root`. This can be solved by using the `--user` flag when running the container, which allows us to specify the uid and gid of the container's user. This way, we can make sure that the container's user has the same uid and gid as the host's user.

Example with default `root` user:

```sh
# run a container which creates a file
$ docker run --rm -v $(pwd):/app alpine sh -c "touch /app/file.txt"
# check the file permissions
$ ls -l file.txt

-rw-r--r-- 1 root root 0 Feb 10 16:39 file.txt
```

Example with custom user:

```sh
# if you are following along, you'll need to delete the file generated previously
$ sudo rm file.txt
# run a container which creates a file with the current user. $(id -u) and $(id -g) are used to get the current user's uid and gid
$ docker run --rm -v $(pwd):/app --user $(id -u):$(id -g) alpine sh -c "touch /app/file.txt"
$ ls -l file.txt

-rw-r--r-- 1 <YOUR_USERNAME> <YOUR_GROUP> 0 Feb 10 16:41 file.txt
```

# SurrealDB issue

[This issue](https://github.com/surrealdb/surrealdb/issues/3468) commented a problem with file permissions when running the container. Here's the excerpt:

```sh
$ docker run --rm --pull always -p 8000:8000 -v /mydata:/mydata surrealdb/surrealdb:latest start file:/mydata/mydatabase.db

2024-02-09T13:41:36.643952Z  INFO surreal::env: Running 1.1.1+20240116.b261047 for linux on x86_64
2024-02-09T13:41:36.643985Z  WARN surreal::dbs: ❌🔒 IMPORTANT: Authentication is disabled. This is not recommended for production use. 🔒❌
2024-02-09T13:41:36.644012Z  INFO surrealdb::kvs::ds: Starting kvs store at file:///mydata/mydatabase.db
2024-02-09T13:41:36.645252Z  INFO surrealdb::kvs::ds: Started kvs store at file:///mydata/mydatabase.db
2024-02-09T13:41:36.645267Z ERROR surreal::cli: There was a problem with the database: There was a problem with a datastore transaction: Failed to create RocksDB directory: `Os { code: 13, kind: PermissionDenied, message: "Permission denied" }`.
```

The problem here is that the container is trying to create a file inside the `/mydata` directory, but it doesn't have the permissions to do so. To know exactly what the problem is, we would need to know the permissions of the `/mydata` directory on the host, and the user used in the container.

Checking out the permissions of the `/mydata` directory is easy

```sh
# run the container, expect it to fail as the user mentioned
$ docker run --rm --pull always -p 8000:8000 -v /mydata:/mydata surrealdb/surrealdb:latest start file:/mydata/mydatabase.db
# check the permissions of the /mydata directory
$ ls -la /mydata
total 8
drwxr-xr-x  2 root root 4096 Feb 10 16:59 ./
drwxr-xr-x 20 root root 4096 Feb 10 16:59 ../
```

Here we can see that the `/mydata` directory is owned by `root`, and it has read, write, and execute permissions for the owner, and read and execute permissions for the group and execute for everyone else. This means that the container needs to run as `root` in order to create files inside the `/mydata` directory. Let's test that out:

```sh
$ docker run --rm --pull always --user root -p 8000:8000 -v /mydata:/mydata surrealdb/surrealdb:latest start file:/mydata/mydatabase.db

latest: Pulling from surrealdb/surrealdb
Digest: sha256:7d00d1b15ccfefdd744c3640122b76fc56eb7dc06d6eebfa3728a28eb1bbaa69
Status: Image is up to date for surrealdb/surrealdb:latest

 .d8888b.                                             888 8888888b.  888888b.
d88P  Y88b                                            888 888  'Y88b 888  '88b
Y88b.                                                 888 888    888 888  .88P
 'Y888b.   888  888 888d888 888d888  .d88b.   8888b.  888 888    888 8888888K.
    'Y88b. 888  888 888P'   888P'   d8P  Y8b     '88b 888 888    888 888  'Y88b
      '888 888  888 888     888     88888888 .d888888 888 888    888 888    888
Y88b  d88P Y88b 888 888     888     Y8b.     888  888 888 888  .d88P 888   d88P
 'Y8888P'   'Y88888 888     888      'Y8888  'Y888888 888 8888888P'  8888888P'


2024-02-10T20:01:53.012007Z  INFO surreal::env: Running 1.1.1+20240116.b261047 for linux on x86_64
2024-02-10T20:01:53.012027Z  WARN surreal::dbs: ❌🔒 IMPORTANT: Authentication is disabled. This is not recommended for production use. 🔒❌
2024-02-10T20:01:53.012041Z  INFO surrealdb::kvs::ds: Starting kvs store at file:///mydata/mydatabase.db
2024-02-10T20:01:53.063538Z  INFO surrealdb::kvs::ds: Started kvs store at file:///mydata/mydatabase.db
2024-02-10T20:01:53.063729Z  INFO surrealdb::node: Started node agent
2024-02-10T20:01:53.064001Z  INFO surrealdb::net: Started web server on 0.0.0.0:8000
^C2024-02-10T20:01:56.798021Z  INFO surrealdb::net: SIGINT received. Waiting for graceful shutdown... A second signal will force an immediate shutdown
2024-02-10T20:01:56.798094Z  INFO surrealdb::node: Gracefully stopping node agent
2024-02-10T20:01:56.798101Z  INFO surrealdb::node: Stopped node agent
2024-02-10T20:01:56.798124Z  INFO surrealdb::net: Web server stopped. Bye!
```

The container ran successfully as root!

## The twist

You might be asking yourself _Isn't there an alternative to running the container as root?_. The answer is yes! and we need to take a closer look at our `/mydata` directory in our host machine. Do you remember creating it? No! That's because the docker daemon created it for you when you ran `docker run -v ...`. By default, docker will create the folder with `root` user and group. We can change this by previously creating the folder with the correct user and group.

```sh
# if you are following along, you'll want to delete the folder generated previously
$ sudo rm -rf /mydata
# Since / is owned by root, I suggest we use our current folder to store the data
$ mkdir ./mydata
# Check that the folder is owned by your current user
$ ls -la ./mydata
total 8
drwxrwxr-x  2 <YOUR_USERNAME> <YOUR_GROUP> 4096 Feb 10 17:08 ./
drwxrwxr-x 10 <YOUR_USERNAME> <YOUR_GROUP> 4096 Feb 10 17:08 ../

# run the container without root
$ docker run --rm --pull always -p 8000:8000 -v $(pwd)/mydata:/mydata surrealdb/surrealdb:latest start file:/mydata/mydatabase.db
# various logs
2024-02-10T20:10:25.453868Z ERROR surreal::cli: There was a problem with the database: There was a problem with a datastore transaction: Failed to create RocksDB directory: `Os { code: 13, kind: PermissionDenied, message: "Permission denied" }`.
```

But wait, I got an error! That's because the container is not running with the same user and group as the host. We can fix this by running the container with the same user and group as the host.

```sh
$ docker run --rm --pull always --user $(id -u):$(id -g) -p 8000:8000 -v $(pwd)/mydata:/mydata surrealdb/surrealdb:latest start file:/mydata/mydatabase.db
# happy logs :D
```

## Going a bit deeper

If you've got experience with Docker, you might be thinking that it's weird that you havent' gotten to this problem earlier with other containers. There are 2 reasons for this:

- usually, containers run with `root` user, giving the container full access over your files
- when the containers run with a different user, they usually run with the default user that has the same uid and gid as the host's user (`1000:1000` in unix systems)

The problem in the `surrealdb` image is that it doesn't follow the second point. We can check what user and group the container is running with by running the `id` command inside the container. We can do this by creating a new Dockerfile that copies the `id` command from a busybox image and running it inside the container.

```Dockerfile
# file name: Dockerfile
FROM surrealdb/surrealdb:latest

# Install shell and utilities, like the `id` command
COPY --from=busybox:1.35.0-uclibc /bin/* /bin/
```

Then we build the image and run the `id` command inside the container with `-u` and `-g` flags to get the user and group.

```sh
$ docker build . -t surreal
$ docker run -ti --entrypoint id surreal -u
65532
$ docker run -ti --entrypoint id surreal -g
65532
```

The `surrealdb` container is running with user and group `65532:65532`. If we were to create another image and add `USER 1000` to the Dockerfile, we wouldn't need to specify the `--user` flag when running the container (if we are using the default `1000:1000` user in the host).

# Conclusion

In this post, we've learned about file permissions in Linux and Docker containers. We've seen how the file permissions are managed by the host's kernel, and how the container's user can affect the file permissions. We've also seen how to solve file permission issues by running the container with the correct user and group. I hope this post has been helpful, and that you now have a better understanding of file permissions in Docker containers
