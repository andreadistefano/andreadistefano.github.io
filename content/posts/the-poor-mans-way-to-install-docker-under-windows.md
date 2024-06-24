---
title: "The poor man's way to Install Docker under Windows (without Docker Desktop)"
date: 2023-07-31T17:12:08+02:00
draft: false
showToc: true
tags: ["docker", "windows", "wsl2", "how-to"]
---

This is, to the best of my knowledge, the optimal way to install Docker under Windows if you don't have a Docker Desktop license.
<!--more-->
## The Problem
You need Docker on your Windows work machine, but you:
* don't have access to a Docker Desktop license;
* don't want to buy one;
* work for someone who doesn't want to buy one for you either.

## Disclaimer

**I AM NOT A LAWYER, AND THIS IS NOT LEGAL ADVICE.**

Docker Desktop is licensed under the [Docker Desktop license agreement](https://docs.docker.com/subscription/desktop-license/), which makes it "*free for small businesses (fewer than 250 employees AND less than $10 million in annual revenue), personal use, education, and non-commercial open source projects.*"

Docker Engine and the Docker CLI are licensed under the [Apache License 2.0](https://github.com/docker/cli/blob/master/LICENSE).

To me, that means that if you work for a business with more than 250 employees or more than $10 million in annual revenue you are not allowed to use Docker Desktop, but you can still use the Docker Engine and the Docker CLI.

Also, if you follow this guide and bad things happen (you get sued, your computer explodes, you lose the love and trust of your friends and family), it's not my fault.

## The Solution

We will
* install WSL2;
* install `docker` inside WSL2;
* configure it as a `systemd` service;
* use `docker` inside WSL to build the cli executable for Windows.

This will allow us to use the `docker` command from PowerShell, VS Code or any other tool that relies on the existence of `docker` on the Windows `PATH`.

### Prerequisites

1. Go grab a cup of coffee, tea, or your favorite beverage.
2. Install [PowerShell 7](https://learn.microsoft.com/en-us/powershell/scripting/install/installing-powershell-on-windows)[^1].
3. Install Windows Terminal from the [Microsoft Store](https://apps.microsoft.com/store/detail/windows-terminal/9N0DX20HK701) or [GitHub](https://github.com/microsoft/terminal/releases/latest) (optional, but highly recommended if you want to keep your sanity instead of having to manage multiple console windows at the same time).
4. Set PowerShell 7 as the default shell in Windows Terminal (again, optional but highly recommended).
5. Install [WSL2](https://learn.microsoft.com/en-us/windows/wsl/).
6. Install Ubuntu 22.04 from the [Microsoft Store](https://www.microsoft.com/en-us/p/ubuntu-2004-lts/9n6svws3rx71?rtc=1&activetab=pivot:overviewtab), or from this [direct link](https://wslstorestorage.blob.core.windows.net/wslblob/Ubuntu2204-221101.AppxBundle).
7. Enable `systemd` in WSL2:
    1. `sudo -e /etc/wsl.conf`
    2. Add
        ```
        [boot]
        systemd=true
        ```
    3. Save and exit.
    4. Restart WSL2 from PowerShell: `wsl --shutdown`.

### Install Docker
1. Install [Docker](https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository) inside WSL2.
2. Add your user to the `docker` group: `sudo usermod -aG docker $USER`.
3. Override the `docker.service` file: `sudo systemctl edit docker` adding the following text

    ```
    [Service]
    ExecStart=
    ExecStart=/usr/bin/dockerd -H fd:// -H tcp://0.0.0.0:2375 --containerd=/run/containerd/containerd.sock
    ```
   between
   
    `### Anything between here and the comment below will become the new contents of the file`

   and

   `### Lines below this comment will be discarded`

   
    In this way, Docker will listen on a TCP socket on port `2375` (feel free to change it if you want) and we will be able to connect to the Docker Engine from Windows.
6. Reload the `systemd` daemon: `sudo systemctl daemon-reload`.
7. Restart the Docker service: `sudo systemctl restart docker.service`.
8. Enable the Docker service to start automatically: `sudo systemctl enable docker.service`.
9. Optionally, you can install `keychain` to make sure you WSL will not be killed when you close the last shell window[^2]:
    1. `sudo apt update && sudo apt install keychain -y`
    2. `sudo -e /etc/profile.d/keep_wsl_running.sh`
    3. Paste this
        ```
        #!/usr/bin/env sh
        eval $(keychain -q)
        ```
    4. Exit from WSL.
    5. Start a new WSL shell.
    6. From now on, when you really want Windows to kill the WSL session, use the command `keychain -k all` before closing all the WSL windows.

### Build the `docker` CLI

This is the worst, but I haven't found a way to avoid it.

1. Install `git`: `sudo apt install git`.
2. Clone the docker cli repository: `git clone  --depth 1 --branch v24.0.4  https://github.com/docker/cli && cd cli`.
3. `docker context create build_windows`.
4. `docker buildx create build_windows`.
5. `docker buildx bake --set binary.platform=windows/amd64`.
6. While you wait for the build to end, think about your life and the choices that brought you here. The build takes a couple of minutes to complete on my machine.
7. Your new `docker-windows-amd64.exe` will be in the `build` folder of the repository.
8. Create a directory under C: (or wherever you want) called `docker-cli`: `mkdir /mnt/c/docker-cli`.
9. Move `docker-windows-amd64.exe` to that folder: `mv build/docker-windows-amd64.exe /mnt/c/docker-cli/docker.exe`.
10. Add `C:\docker-cli` to your system `PATH` environment variable.
11. Create a new environment variable called `DOCKER_HOST` with the value `tcp://localhost:2375`.
12. Close all your Windows Terminal windows and start a new one.
12. If everything works, you should be able to run `docker version` in PowerShell and see the output of the Docker Engine running inside WSL2.

[^1]: I can hear some of you saying "What do you mean, install PowerShell?". I had the same reaction. It turns out, the default Windows PowerShell is not the same as the new PowerShell 7. I'll have a blog post here at some point about that.

[^2]: Thanks to [this answer on AskUbuntu](https://askubuntu.com/a/1436045/790111) for the tip.
