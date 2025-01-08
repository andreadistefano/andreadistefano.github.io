---
title: "Printing Github Secrets"
date: 2025-01-08T16:51:40+01:00
draft: false
showToc: true
tags: ["github", "github-actions", "how-to"]
---

Happy new year! 🎉 \
Let's start by doing something YOU SHOULD NEVER EVER DO.

<!--more-->

## The Problem

At `$DAYJOB`, a colleague that is no longer working with us stored a useful piece of information in a GitHub secret and nowhere else. We usually also store these values in a team-shared password manager, so they can always be retrieved if needed, but apparently this one was missed. \
We need a way to retrieve the secret. Fortunately, we have access to the repository where it is stored.

## Disclaimer
This will output the secret in plain text in the GitHub Actions log, which usually is a **VERY BAD IDEA**. I don't know for how long the logs are retained (a web search is telling me 90 days). I don't know if the logs are still retained even after the workflow run is deleted.

## The Solution
We'll use a GitHub Actions workflow that prints the secret to the log.

1. Create a new branch in the repository where the secret is stored, name it something like `ci_secret`.
2. Create a new file in the `.github/workflows` directory, name it something like `secret.yml`.
3. Add the following content to the file:
    ```yaml
    name: Secret

    on:
    workflow_dispatch:
    push:
      branches:
        - ci_secret

    jobs:
      secret:
        runs-on: ubuntu-latest

        steps:
          - name: print_secret
            run: |
              echo ${{ secrets.YOUR_SECRET_NAME }} | sed 's/./& /g'
    ```

    Remember to replace `YOUR_SECRET_NAME` with the name of the secret you want to print. You can also add multiple `echo` commands to print multiple secrets. \
    The `sed` command adds a space between each character of the secret, so that GitHub does not replace the value with `***`.

4. Commit the changes and push the branch to the repository.

5. See the logs of the workflow run in the GitHub Actions tab of the repository.

6. Delete the workflow run and the branch after you've retrieved the secret.

7. If possible, rotate the secret.
