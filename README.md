# Example Voting App

A simple distributed application running across multiple Docker containers.

## Getting started

Download [Docker Desktop](https://www.docker.com/products/docker-desktop) for Mac or Windows. [Docker Compose](https://docs.docker.com/compose) will be automatically installed. On Linux, make sure you have the latest version of [Compose](https://docs.docker.com/compose/install/).

This solution uses Python, Node.js, .NET, with Redis for messaging and Postgres for storage.

Run in this directory to build and run the app:

```shell
docker compose up
```

The `vote` app will be running at [http://localhost:8080](http://localhost:8080), and the `results` will be at [http://localhost:8081](http://localhost:8081).

Alternately, if you want to run it on a [Docker Swarm](https://docs.docker.com/engine/swarm/), first make sure you have a swarm. If you don't, run:

```shell
docker swarm init
```

Once you have your swarm, in this directory run:

```shell
docker stack deploy --compose-file docker-stack.yml vote
```

## Run the app in Kubernetes

The folder k8s-specifications contains the YAML specifications of the Voting App's services.

Run the following command to create the deployments and services. Note it will create these resources in your current namespace (`default` if you haven't changed it.)

```shell
kubectl create -f k8s-specifications/
```

The `vote` web app is then available on port 31000 on each host of the cluster, the `result` web app is available on port 31001.

To remove them, run:

```shell
kubectl delete -f k8s-specifications/
```

## Architecture

![Architecture diagram](architecture.excalidraw.png)

* A front-end web app in [Python](/vote) which lets you vote between two options
* A [Redis](https://hub.docker.com/_/redis/) which collects new votes
* A [.NET](/worker/) worker which consumes votes and stores them in…
* A [Postgres](https://hub.docker.com/_/postgres/) database backed by a Docker volume
* A [Node.js](/result) web app which shows the results of the voting in real time

## Container hardening

The upstream project ships Dockerfiles that use unpinned base-image tags and run as root. This fork changes that:

* Added a `.dockerignore` to `vote/`, `result/` and `worker/` (excludes `.git`, `node_modules`, `*.md` and `.env` from the build context).
* Pinned each base image by digest, keeping the same versions as upstream (Python 3.11, Node 18, .NET 7). Node 18 and .NET 7 are past end of life, so image scanners will likely report vulnerabilities in them.
* Run every service as a non-root user: `vote` and `worker` as `app` (uid 10001), `result` as the built-in `node` user (uid 1000). Check with `docker run --rm vote id`, or `docker run --rm --entrypoint id result` for the others.
* A non-root user cannot bind port 80, so `vote` now listens on 8000 and `result` on 3000. `docker-compose.yml` and `seed-data/generate-votes.sh` were updated to match. The URLs on your machine are unchanged: vote at 8080, result at 8081.
* Not updated: `k8s-specifications/` still assumes port 80 for these services.
## CI pipeline

`.github/workflows/ci.yml` runs on every push. It builds `vote`, `result` and `worker` as a matrix, then pushes each image to GitHub Container Registry as `ghcr.io/<owner>/voting-<service>:<commit sha>`. Images are tagged with the commit SHA, never `latest`.

* **Test:** the only test is a basic syntax check of the vote app (`python -m py_compile vote/app.py`). It catches syntax errors, not behaviour. There is no real test suite.
* **Scan:** Trivy scans each image for HIGH and CRITICAL vulnerabilities. It is set to report only (`exit-code: '0'`) so findings do not fail the build. Node 18 and .NET 7 are past end of life, so `result` and `worker` report the most findings.
* **Auth:** the push uses the `GITHUB_TOKEN` that GitHub creates for each run, so no secret is stored in the repository.
* **Action version:** Trivy runs as `aquasecurity/trivy-action@v0.36.0`. Version 0.28.0 no longer runs because a dependency tag it needs has been removed upstream.
* **Removed:** the three upstream `call-docker-build-*` workflows were deleted in favour of this one.
## Notes

The voting application only accepts one vote per client browser. It does not register additional votes if a vote has already been submitted from a client.

This isn't an example of a properly architected perfectly designed distributed app... it's just a simple
example of the various types of pieces and languages you might see (queues, persistent data, etc), and how to
deal with them in Docker at a basic level.
