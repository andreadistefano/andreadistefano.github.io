---
title: "Docker Multi-Arch Builds on Github Actions"
date: 2025-01-14T15:45:00+01:00
draft: false
---

In the last few months, I've been doing some work on building Docker images for multiple architectures (at the moment `linux/amd64` and `linux/arm64`) on Github Actions. \
Here is how I did it.

<!--more-->

## The Problem
[Some time ago]({{< ref "instrumenting-java-with-newrelic.md" >}}) I wrote about how I needing to build multi-arch Docker images. \
We have some different images to build using different technologies (Java, Python, JS, and others), but all of them need to be built for `linux/amd64` and `linux/arm64`.

When I joined `$DAYJOB`, all the builds were being done using Docker's own [build-push-action](https://github.com/docker/build-push-action). It works great and has a very convenient `platforms` argument, which allows you to build for multiple architectures ([here](https://docs.docker.com/build/ci/github-actions/multi-platform/) is a nice example on Docker docs). Building for multiple architectures in this way requires setting up QEMU and running the build in an emulated environment.

This works great for many cases (e.g. a Python application where you just copy the source files to the image and run `pip install -r requirements.txt`), but it can become slow for others (e.g. a Javascript app which needs `webpack` to build the assets). \
By *slow*, I mean that the build time can grow from a few minutes to literally **HOURS**.

## The Solution
What I came up with is running the builds on 2 GitHub-hosted runners, one for each architecture, leveraging the GitHub Actions `matrix`, which lets you run multiple steps of your build in parallel. \
Here is a basic workflow file that runs when a tag is pushed to the repository:

```yaml
name: Build and Push Docker Image

on:
  push:
    tags:
      - '*'

env:
  IMAGE_NAME: my-organization/image-name
  CACHE_NAME: my-organization/cache:image-name

jobs:
  docker-build:
    name: docker-build
    runs-on: ${{ matrix.runner_platform.runner }}
    strategy:
      matrix:
        runner_platform: [ { runner: amd64_runner, platform: linux/amd64, architecture: amd64 }, { runner: arm64_runner, platform: linux/arm64, architecture: arm64 } ]

    steps:
      - name: "Checkout ${{ github.ref }} ( ${{ github.sha }} )"
        uses: actions/checkout@v4
        with:
          persist-credentials: false
          fetch-depth: 0

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Build and push Docker Image
        id: docker_build
        uses: docker/build-push-action@v6
        with:
          context: .
          platforms: ${{ matrix.runner_platform.platform }}
          cache-from: type=registry,ref=${{ env.CACHE_NAME }}-${{ matrix.runner_platform.architecture }}
          cache-to: type=registry,ref=${{ env.CACHE_NAME }}-${{ matrix.runner_platform.architecture }}
          push: true
          outputs: type=image,name=${{ env.IMAGE_NAME }},push-by-digest=true,name-canonical=true,push=true

      - name: Export digest
        run: |
          mkdir -p /tmp/digests/${{ matrix.runner_platform.architecture }}/
          digest="${{ steps.docker_build.outputs.digest }}"
          touch "/tmp/digests/${{ matrix.runner_platform.architecture }}/${digest#sha256:}"

      - name: Upload digest
        uses: actions/upload-artifact@v4
        with:
          name: digests-${{ matrix.runner_platform.architecture }}
          path: /tmp/digests/*
          if-no-files-found: error
          retention-days: 1

  merge_docker_images:
    needs: [ docker-build ]
    runs-on: ubuntu-latest
    steps:
      - name: Download digests
        uses: actions/download-artifact@v4
        with:
          path: /tmp/digests
          pattern: digests-*
          merge-multiple: true

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v3

      - name: Login to Docker Hub
        uses: docker/login-action@v3
        with:
          username: ${{ secrets.DOCKERHUB_USERNAME }}
          password: ${{ secrets.DOCKERHUB_TOKEN }}

      - name: Create manifest list for ${{ env.IMAGE_NAME }} and push
        working-directory: /tmp/digests
        run: |
          digest_amd64=$(ls amd64)
          digest_arm64=$(ls arm64)
          docker buildx imagetools create \
          --tag ${{ env.IMAGE_NAME }}:latest \
          --tag ${{ env.IMAGE_NAME }}:${{ github.ref_name }} \
          ${{ env.IMAGE_NAME }}@sha256:${digest_amd64} \
          ${{ env.IMAGE_NAME }}@sha256:${digest_arm64}

      - name: Display Docker Image Tag
        run: "echo Docker Image Tag: ${{ env.IMAGE_NAME }}:${{ github.ref_name }}"
```

Some things to note:
* The `matrix` strategy is used to run the build on 2 different runners, one for each architecture. Using a dictionary for `runner_platform` as an input to the `matrix` allows you to define the name of the runner, the platform, and architecture for each.
* The `docker-build` job builds the image, pushes it to Docker Hub and uploads the digest to an artifact.
* The `merge_docker_images` job downloads the artifact that contains digests of both the images, creates a single manifest list containing both, and pushes it to Docker Hub.

---

This simple hack landed me my current job, hopefully it helps you too.

Until next time!