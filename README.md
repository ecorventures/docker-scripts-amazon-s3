# ecor/scripts-s3

This is a [data volume container](https://docs.docker.com/engine/userguide/containers/dockervolumes/)
that includes several bash scripts for interacting with Amazon S3. These scripts
can be attached to any container capable of running a bash script.

## Preparing the volume

First, the scripts need to be registered with the Docker daemon:

```sh
docker create \
  --name scripts-s3 \
  -e "AWS_KEY=AKIAIGHI3JKL7MNOPQ2R" \
  -e "AWS_SECRET=AbCDEfgHIJ1kLMN2O1PQRsTUvwxyZABcdEFGHi9J" \
  ecor/scripts-s3 /bin/true
```

Running `docker ps -a` should show something like:

```sh
CONTAINER ID   IMAGE            COMMAND        CREATED          STATUS   PORTS    NAMES
01d5468ac4cf   ecor/scripts-s3  "/bin/true"    15 minutes ago   Created           scripts-s3
```

Once the volume is created, it can be used by other containers.

### How do I access the scripts from another container?

Using the `--volumes-from` flag, you can attach these scripts to any container.
For example, connecting to a golang environment would look like:

```sh
docker run -i -t -d --name go \
  -v /path/to/my/go/code:/app \
  -w /app \
  --volumes-from scripts-s3 \
  golang:1.6.0-wheezy bash
```

`ecor/scripts-s3` exposes a volume called `/scripts/aws/s3`, which will be
accessible _inside_ the `go` container.

It's now possible to open a TTY session and run the scripts by hand:

```sh
docker exec -i -t go bash
```

^^ This opens a bash shell in the `go` container.

```sh
root@go:/app# ls -l /scripts/aws/s3/
total 28
-rwxr-xr-x 1 root root  567 Sep 24  2007 Licence
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
  --volumes-from scripts-s3 \
  postgres /scripts/aws/s3/s3-put <params>
```

The example above requires parameters passed to the s3 script, but the point is
a script can be executed arbitrarily using docker and existing images. The example
above was taken from a PostgreSQL environment where a daily cronjob executes the
command (from the example) to send a daily Postgre backup file to Amazon S3.
