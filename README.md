# ecor/scripts-aws-s3

[![](https://badge.imagelayers.io/ecor/scripts-aws-s3:latest.svg)](https://imagelayers.io/?images=ecor/scripts-aws-s3:latest 'Get your own badge on imagelayers.io')

This is a [data volume container](https://docs.docker.com/engine/userguide/containers/dockervolumes/)
that includes several bash scripts for interacting with Amazon S3. These scripts
can be attached to any container capable of running a bash script. It is
extremely lightweight. It's designed to attach functionality to other
containers. It is **not** designed to execute S3 commands from the Docker host.
If that's what you want, you are more likely to find [yep1/s3](https://hub.docker.com/r/yep1/s3/)
to be more useful.

## Preparing the volume

First, the scripts need to be registered with the Docker daemon:

```sh
docker create \
  --name scripts-aws-s3 \
  ecor/scripts-aws-s3 /bin/true
```

Running `docker ps -a` should show something like:

```sh
CONTAINER ID   IMAGE                COMMAND        CREATED          STATUS   PORTS    NAMES
01d5468ac4cf   ecor/scripts-aws-s3  "/bin/true"    15 minutes ago   Created           scripts-aws-s3
```

Once the volume is created, it can be used by other containers.

### How do I access the scripts from another container?

Using the `--volumes-from` flag, you can attach these scripts to any container.
For example, connecting to a golang environment would look like:

```sh
docker run -i -t -d --name go \
  -e "AWS_KEY=AKIAIGHI7JKL7MNOPQ2R" \
  -e "AWS_SECRET=AbCDEfgHIJ1kLMN2O1PQRsTUvwxyZABcdEFGHi9J" \
  -v /path/to/my/go/code:/app \
  --volumes-from scripts-aws-s3 \
  golang:1.6.0-wheezy bash
```

`ecor/scripts-aws-s3` exposes a volume called `/scripts/aws/s3`, which will be
accessible _inside_ the `go` container.

It's now possible to open a TTY session and run the scripts by hand:

```sh
docker exec -i -t go bash
```

^^ This opens a bash shell in the `go` container.

```sh
root@go:/app# ls -l /scripts/aws/s3/
total 28
-rwxr-xr-x 1 root root 4085 Mar 02  2016 delete
-rwxr-xr-x 1 root root 4085 Mar 02  2016 get
-rwxr-xr-x 1 root root  567 Sep 24  2007 Licence
-rwxr-xr-x 1 root root 4085 Mar 02  2016 put
-rw-r--r-- 1 root root 9886 Oct 10  2007 s3-common-functions
-rwxr-xr-x 1 root root 3519 Oct 10  2007 s3-delete
-rwxr-xr-x 1 root root 3562 Oct 10  2007 s3-get
-rwxr-xr-x 1 root root 4085 Oct 10  2007 s3-put
```

You can also execute one of these scripts using any existing container that has
bash support. For example, if you have a PostgreSQL image called `postgres`
running on an Ubuntu image, you could just execute an S3 command by running:

```sh
docker run --rm \
  --name db-backup \
  --volumes-from scripts-aws-s3 \
  postgres /scripts/aws/s3/put <params>
```

The example above requires parameters passed to the s3 script, but the point is
a script can be executed arbitrarily using docker and existing images. The example
above was taken from a PostgreSQL environment where a daily cronjob executes the
command (from the example) to send a daily Postgre backup file to Amazon S3.

# Scripts

There are a total of 7 scripts, but only 3 really matter for general use.
Any script (found in the `/lib` directory) beginning with `s3-` is part of the
original [AWS bash commands](http://aws.amazon.com/code/943),
which were a userland implementation of S3 functionality made available via
Amazon. These commands pre-date Docker. There was a slight modification made
to the original `s3-common-functions` file to better support Amazon secrets.

The remaining scripts act as a polyfill to modernize the original scripts
(which happen to work pretty darn well) for use within Docker. It forces use
of SSL (HTTPS) for all communication and handles all credentials automatically.
Remember that these "polyfill" functions are mostly for convenience. If you
have a use case where you need to use a different AWS Key/Secret from the one
supplied in the `docker run` command, use the `s3-____` file directly.

**get**: `get /my.bucket/path/to/file.txt > /path/to/output.txt`

The `get` command reads the contents of the remote file and writes it to
`stdout`. In the example, we've piped `stdout` to a file at `/path/to/output.txt`.

**put**: `put /my/local/file > /my.bucket/path/to/file.txt`.

_You must urlencode /my.bucket/path/to/file.txt yourself!_

By default, all files are considered `plain/text`. If you want to specify an
alternative content type, supply a `-c` option. For example, `-c application/octet-stream`.

**delete**: `delete /my.bucket/path/to/file.txt`.

Removes a file. This does not work on directories unless they're empty (this is
an Amazon S3 thing, not a limitation of these scripts).
