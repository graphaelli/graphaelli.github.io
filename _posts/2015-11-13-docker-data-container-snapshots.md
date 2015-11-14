---
layout: post
title:  "Docker Data Container Snapshots"
date:   2015-11-13 17:10:02
---
It's straightforward to create a data container with pre-baked archives included: `COPY` them on during `docker build` and off you go.  But what if you can't or don't want to create the data ahead of time?

At [RealScout][RealScout], we recently started experimenting with the approach detailed below to create docker images from postgres snapshots for distribution to development environments.  Briefly, it looks like:

1. Create and initialize a data container
1. Start postgres using the volume from that data container
1. Restore data
1. Stop postgres
1. Create an image from the populated data container

## Setup

First, build a data container image, for example with this `Dockerfile`. :
{% highlight docker %}
FROM debian:jessie

ENV PGDATA /var/lib/postgresql/data

# ids must match postgres container
RUN groupadd -r -g 999 postgres && useradd -u 999 -r -g postgres postgres
RUN mkdir -p -m 0700 ${PGDATA} && chown -R postgres:postgres ${PGDATA}

VOLUME ${PGDATA}

CMD /bin/true
{% endhighlight %}

{% highlight console %}
$ docker build .
Successfully built 221c07e4f85c
{% endhighlight %}

Now create a postgres data container using that image:

{% highlight console %}
$ docker run --name postgres-data 221c07e4f85c

$ docker ps -af ancestor=221c07e4f85c
CONTAINER ID        IMAGE               COMMAND                  CREATED             STATUS                              PORTS               NAMES
d13320582ff2        221c07e4f85c        "/bin/sh -c /bin/true"   1 seconds ago       Exited (0) Less than a second ago                       hopeful_boyd
{% endhighlight %}

Now start a postgres server (using an image from Docker Hub here) using the volume from that data container:
{% highlight console %}
$ docker run -d --volumes-from=postgres-data postgres
146ad1bcbb45c74100f5345facff8be2ba9a532cbe0c5642a0c0bf840b4234e4
{% endhighlight %}

Now we can load up some data.  At this point, you should probably scrub out anything from the database backup that you don't want to include in the image.

{% highlight bash %}
pg_restore ...
{% endhighlight %}

## The clever part

We're going to `docker build` inside of a third container that has the data volume mounted inside of it.  It's [not quite docker in docker][jpetazzo], but it's related.

First, take the Dockerfile above and `COPY data/ ${PGDATA}` to it so it looks like:
{% highlight docker %}
FROM debian:jessie

ENV PGDATA /var/lib/postgresql/data

# ids must match postgres container
RUN groupadd -r -g 999 postgres && useradd -u 999 -r -g postgres postgres
RUN mkdir -p -m 0700 ${PGDATA} && chown -R postgres:postgres ${PGDATA}
COPY data/ ${PDATA}

VOLUME ${PGDATA}

CMD /bin/true
{% endhighlight %}

(We just have a single `Dockerfile` used for initial and snapshot builds and an empty `data/` sitting around for use by the initial build.)

Now stick this in `snapshot.sh`:

{% highlight bash %}
{%raw%}
#!/bin/bash -e
BUILD_WD=/tmp/build
DATA_CONTAINER=${1:-postgres-data}
DOCKERFILE=${2:-Dockerfile}

# like --volumes-from, but allow for mounting in a different location
DATA_VOLUME=$(docker inspect --format '{{range $mount := .Mounts}}{{if eq $mount.Destination "/var/lib/postgresql/data"}}{{$mount.Source}}{{end}}{{end}}' $DATA_CONTAINER)

# stop sends TERM, like postgres wants
docker stop -t 60 postgres

# create a docker container with data mounted at /data and run docker build inside it
docker run --rm \
       -v ${PWD}/${DOCKERFILE}:/tmp/build/Dockerfile \
       -v ${DATA_VOLUME}:${BUILD_WD}/data \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v $(which docker):/bin/docker \
       -w ${BUILD_WD} \
       debian:jessie \
       docker build .
{%endraw%}
{% endhighlight %}

if you used the `--name` flag on `docker run` earlier, you can snapshot with:
{% highlight bash %}
./snapshot.sh
{% endhighlight %}

I hope that helps.  Please let me know if I screwed anything up.

[RealScout]: http://realscout.com
[jpetazzo]: http://jpetazzo.github.io/2015/09/03/do-not-use-docker-in-docker-for-ci/
