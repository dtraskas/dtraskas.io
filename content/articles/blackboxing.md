---
author: "Dimitris Traskas"
date: 2019-07-19
linktitle: Blackboxing Your Secrets in Kubernetes
menu:
  main:
    parent: 
next: 
prev: 
title: Blackboxing Your Secrets in Kubernetes
weight: 10
summary: "Using Blackbox is essential in the Kubernetes world. This quick guide gives you an overview of the set up process."
BookToC: true
---

Managing your secrets across services deployed in a Kubernetes cluster is a crucial part of the daily development workflow. When we first started working with Microservices we very quickly realised that although Kubernetes had the right mechanisms our repository setup was lacking. 

## Why do this?

When you write code that interacts with databases, Cloud services on AWS, GCP, Azure or any service that requires authentication you almost always have to think about security. You really don't want to have your secrets leaked accidentally by storing them in your git repositories. If you don't store them in a repo however it becomes very quickly difficult to share those secrets across a team of engineers.

## What is the answer?

The answer to storing secrets securely on a git repo is [GPG](https://gnupg.org/). The [StackExchange](https://stackexchange.com) team has kindly developed a tool that deals with git repo secrets called [BlackBox](https://github.com/StackExchange/blackbox). What BlackBox does is to simplify and secure the entire process of storing secrets in your git repositories without jeopardising any leaks.


## How to set it all up

The [Github repo on BlackBox](https://github.com/StackExchange/blackbox) has all the instructions you need to set this up but the summary is this:

1. Download GPG command line tools from [here](https://www.gnupg.org/download/)
2. Generate a GPG key pair by running this:
    ```
    $ gpg --full-generate-key
    ```
3. Accept the default parameters specified (use a key at last 4096 bits long)
4. Enter user info and a secure passphrase
5. Install blackbox using Homebrew by running:
    ```
    $ brew install blackbox
    ```
6. Initialize blackbox in your repo by running:
    ```
    $ blackbox_initialize
    ```
7. If you need to encrypt a file that has secrets then run:
    ```
    $ blackbox_register_new_file <name of the file>
    ```

If you now want to add or remove keys you can use `blackbox_addadmin` or `blackbox_removeadmin`.

## Development workflow

Every time you need to access a secret file you use Blackbox. You will start editing the file of interest using `blackbox_edit_start <name of the file>` and once done close it with `blackbox_edit_end`. After those changes you can use `git commit` and `git push` to push all your changes without exposing any secrets. That way if you want to source control secrets you are storing in Kubernetes you can very easily follow this process.