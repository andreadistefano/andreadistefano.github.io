---
title: "Instrumenting Java with NewRelic (without the source code)"
date: 2024-12-13T19:04:11+01:00
draft: false
showToc: true
tags: ["how-to"]
---

So, I just spent this Friday afternoon at `$DAYJOB` figuring out how to instrument a Java application (of which we have no source code available) with NewRelic. \
I thought I'd share my findings here, should I ever need to do this again.
<!--more-->

## The Problem
So, at `$DAYJOB`, we run a Java application in a Docker container (actually, on a Kubernetes cluster, but anyway). We're trying to figure out some performance issues in our whole stack, and we already use NewRelic for other monitoring tasks, so why not? \
I'll also point out that the vendor decided to not provide any OpenTelemetry instrumentation in their code (which we would have preferred).

## The Solution
So, here is what I did:
1. I created a new `Dockerfile`. It looks like this:

    ```Dockerfile
    ARG ORIGINAL_TAG="latest"

    FROM curlimages/curl:latest AS downloader

    WORKDIR /home/curl_user
    RUN curl -O https://download.newrelic.com/newrelic/java-agent/newrelic-agent/current/newrelic-java.zip
    RUN unzip newrelic-java.zip
    ADD ./newrelic.yml /home/curl_user/newrelic.yml

    FROM --platform=amd64 <IMAGE>:${ORIGINAL_TAG} AS original

    COPY --from=downloader /home/curl_user/newrelic/newrelic.jar /app
    COPY --from=downloader /home/curl_user/newrelic.yml /app

    WORKDIR /app

    USER root
    RUN mkdir -p /app/logs && chown appuser -R /app/logs

    USER appuser

    ENV APP_JAVA_OPS="-javaagent:/app/newrelic.jar"

    ENTRYPOINT ["/bin/bash", "/app/runserver.sh"]
    ```

    Basically, it uses `curl` to download the NewRelic Java agent and copies it into the final image, along with the NewRelic configuration file (which is stored in the Git repository alongside the `Dockerfile`). As you can see, I just added the `-javaagent:/app/newrelic.jar` to the `APP_JAVA_OPS` environment variable. (Your mileage may vary here, depending on how your application is started.) \
    Some things to note:
    * The `/app/runserver.sh` script is the entrypoint for the container and starts the Java application. I don't need or want to know what's in there.
    * I injected the `ORIGINAL_TAG` argument to the `Dockerfile`, and then I rebuilt the image with the same tag as the original image.
    * I used the `--platform=amd64` flag, because I also need to build this image for ARM64 and the vendor does not provide an `arm64` version of their image. (In practice, I have another `Dockerfile` for ARM64 that looks almost the same, but also uses an ARM64 Java base image.)[^1]

2. I created a `newrelic.yml` file that looks like this:

    ```yaml
    common: &default_settings
      license_key: "NEWRELIC_LICENSE_KEY"
      app_name: "APP_NAME"
      # lots of other things here, check the NewRelic documentation if you're interested
    ```

    I used [this excellent GitHub action](https://github.com/fjogeleit/yaml-update-action) to replace the placeholders with the actual values, stored in GitHub secrets, before building the Docker image. \
    The GitHub action step looks like this:

    ```yaml
    - name: Update newrelic.yml
      uses: fjogeleit/yaml-update-action@main
      with:
        valueFile: 'newrelic.yml'
        changes: |
          {
            "common.license_key": "${{ secrets.NEWRELIC_LICENSE_KEY }}"
            "common.app_name": "${{ secrets.APP_NAME }}"
          }
        commitChange: false
    ```

3. There is no step 3. That's it. The Java application is now instrumented with NewRelic.

Thanks for reading. I hope this helps someone else in the future.

[^1]: I've actually been doing some work on multi-platform Docker builds, and I'll probably write a blog post about that soon.