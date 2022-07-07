---
layout: post
title: "GitHub - Source Repository Migations"
date: 2021-01-28 22:29:01 +00
categories: github
tags:  migration
---

I have been looking for a way to migrate a private Git repository to GitHub without losing commit history.

## Using GIT Command Line Interface

1. Create an empty GitHub repository.

2. Clone the remote repository with the `--bare` option to obtain the clone with no working directory.

    ```bash
    git clone --bare https://<GIT_SERVER_FQDN>/<USER_NAME>/<REPO_NAME>.git
    ```

    >The `-- bare` option represents that a repository will be cloned with the project’s **history**, which can be **pushed** and **pulled** from but not directly modified.
    {: .prompt-info }

3. Push the locally cloned repository to GitHub using the `--mirror` option.

    ```bash
    cd <REPO_NAME>.git
    git push --mirror https://github.com/<USER_NAME>/<REPO_NAME>.git
    ```

    > This ensures that all the references to the cloned repository, such as `branches` and `tags`, are pushed.
    {: .prompt-info}

## Using Shell Script

In order to facilitate any future needs of going through similar migrations I wrote a trivial shell script which bundles all the commands in the [GIT CLI](#using-git-command-line-interface) together into a single executable script.

```bash
#!/bin/bash

read -p "Enter your orginial Git repository URL : " repo
read -p "Enter target GitHub repository URL : " newrepo

IFS='/' 
read -a array <<< "$repo"

reponame = ${array[1]}

git clone --bare $repo

cd $reponame

echo $reponame

git push --mirror $newrepo

cd ..
rm -rf $repo

echo "Done!"
```
