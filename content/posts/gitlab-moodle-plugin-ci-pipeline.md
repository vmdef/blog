---
title: "Gitlab Moodle Plugin Ci Pipeline"
date: 2023-07-24T11:49:38+01:00
draft: true
---

## Introduction

As the Communities Engagement Team diligently works to improve the [Moodle plugins directory](https://moodle.org/plugins), it becomes necessary to make modifications to the main plugin of the site.

To maintain plugin quality, implementing an automated testing pipeline has become essential for developers. In this blog post, we will explore how to set up an automated testing pipeline for Moodle plugins using GitLab CI/CD and the [moodle-plugin-ci](https://moodlehq.github.io/moodle-plugin-ci/) tool. This pipeline will help you automatically build, test, and validate your Moodle plugins, making the development process smoother and more efficient.

## Prerequisites

Before we dive into setting up the pipeline, you'll need the following prerequisites:

* A Moodle plugin hosted on a GitLab repository.
* A GitLab account with sufficient access to the repository.
* Familiarity with Git and basic understanding of GitLab CI/CD.

## Setting up the GitLab Pipeline

### Step 1: Create a GitLab CI/CD Configuration

GitLab CI/CD allows you to define a pipeline that automatically builds, tests, and deploys your code. To set up the CI/CD configuration for your Moodle plugin, create a *.gitlab-ci.yml* file in the root of your repository. This file will define the stages, jobs, and scripts for your pipeline.

### Step 2: Defining Stages, base image and variables

The *gitlab-ci.yml* file outlines the different stages of the pipeline. In the provided example, we see stages like build and test. The **build** stage specifies the prerequisites and dependencies required for the subsequent stages, install moodle-plugin-ci and prepares it for testing. The **test** stage runs a set of tools, thoroughly evaluating the plugin against Moodle's coding guidelines and compatibility standards.

As default Docker image for our environment we use [moodlehq/moodle-php-apache:8.0](https://github.com/moodlehq/moodle-php-apache).

Other services required are MySQL, and Selenium in case we want to run Behat tests.

Most of the variables are used by the Docker containter or moodle-plugin-ci to set up the environment. Moodle uses npm as node as package manages, so it must be available before installing moodle-plugin-ci.

```sh

stages:
- build
- test

default:
image: moodlehq/moodle-php-apache:8.0

services:
- mysql:8
- name: selenium/standalone-chrome:3
    alias: selenium-standalone-chrome

variables:
MOODLE_REPO:
    description: "Moodle code repository (moodle-plugin-ci)."
    value: "https://github.com/moodle/moodle.git"
MOODLE_BRANCH:
    description: "Moodle branch (moodle-plugin-ci)."
    value: "MOODLE_402_STABLE"
MOODLE_BEHAT_WDHOST:
    description: "Selenium Web Driver (moodle-plugin-ci)."
    value: "http://selenium-standalone-chrome:4444/wd/hub"
MYSQL_ALLOW_EMPTY_PASSWORD:
    description: "Allow the container to be started with a blank password for the root user."
    value: "true"
DB:
    description: "Database to be use (moodle-plugin-ci). "
    value: "mysqli"
NODE_VERSION_INSTALL:
    description: "Node version to install in the container."
    value: "16.20.1"
NVM_DIR:
    description: "nvm path"
    value: "/usr/local/nvm"

workflow:
rules:
    - if: $CI_PIPELINE_SOURCE == 'merge_request_event'
```

### Step 3: Build Stage

In this stage, we set up the required dependencies for our Moodle plugin, including installing Node.js, configuring the database, and initializing moodle-plugin-ci. The script also installs moodle-plugin-ci using Composer and creates a symbolic link for easy access.

Note the addition of `export CXXFLAGS="--std=c++14"`. This is a Makefile variable for the c++ compiler. This arises because there is code only supported since C++14 and *node-sass* specifies `-std=c++11` (see [nodejs/node#38367](https://github.com/nodejs/node/issues/38367)).

It's important to note that running *moodle-plugin-ci install* in the root directory of the container should be avoided as it triggers *npm* and causes a "Tracker IdealTree" error to occur.

```sh
moodle-plugin-ci setup:
stage: build
script:
- |
    cd ~
    apt update && apt install -y -q --no-install-recommends \
    openjdk-11-jre-headless \
    git-core \
    default-mysql-client \
    python2 \
    && apt clean
# Install node
- |
    mkdir $NVM_DIR || true
    curl https://raw.githubusercontent.com/creationix/nvm/master/install.sh | bash \
        && . $NVM_DIR/nvm.sh \
        && nvm install $NODE_VERSION_INSTALL
    export NODE_PATH=$NVM_DIR/versions/node/v$NODE_VERSION_INSTALL/lib/node_modules
    export PATH=$NVM_DIR/versions/node/v$NODE_VERSION_INSTALL/bin:$PATH
- |
    if [[ "$DB" == "mysqli" ]]; then
        export DB_HOST="mysql"
    elif [[ "$DB" == "pgsql" ]]; then
        export DB_HOST="postgres"
    else
        export DB_HOST="$DB"
    fi
- |
    export IPADDRESS=`grep "${HOSTNAME}$" /etc/hosts |awk '{print $1}'`
    export MOODLE_BEHAT_WWWROOT="http://${IPADDRESS}:8000"
    export MOODLE_START_BEHAT_SERVERS="NO"
- pushd "/var/www/html"
- rm -rf ci
- curl -sS https://getcomposer.org/installer | php -- --filename=composer --install-dir=/usr/local/bin
- composer create-project -n --prefer-dist --stability=dev moodlehq/moodle-plugin-ci ci ^4
- ln -s `pwd`/ci/bin/moodle-plugin-ci /usr/local/bin
- popd
- . $NVM_DIR/nvm.sh
- cd $CI_PROJECT_DIR/..
- export CXXFLAGS="--std=c++14"
- moodle-plugin-ci install --plugin ./moodle-local_plugins --db-host="$DB_HOST" --no-init -vvv
- cd moodle
```

### Step 4: Test stage

In the test stage, *moodle-plugin-ci* performs a series of checks, including PHP linting, copy-paste detection, code mess detection, validation, and more. These tests ensure that the plugin adheres to coding standards and follows best practices.

There is another job that runs PHPUnit tests for the plugin. It initializes the PHPUnit environment and executes the tests. The pipeline checks for the existence of PHPUnit test files before proceeding with this stage.

Finally, the pipeline runs PHPUnit and Behat tests if they are defined. It verifies that the plugin functions as expected and integrates well with Moodle.

```sh
Analysis:
stage: test
script:
    - moodle-plugin-ci phplint
    - moodle-plugin-ci phpcpd
    - moodle-plugin-ci phpmd
    - moodle-plugin-ci validate
    - moodle-plugin-ci savepoints
    - moodle-plugin-ci mustache
    - moodle-plugin-ci grunt --max-lint-warnings 0
    - moodle-plugin-ci codechecker --max-warnings 0
    - moodle-plugin-ci phpdoc

PHPUnit tests:
stage: test
script:
    - cd moodle
    - php admin/tool/phpunit/cli/init.php
    - php admin/tool/phpunit/cli/util.php --buildcomponentconfigs
    - php -S ${IPADDRESS}:8000 -t $CI_PROJECT_DIR/../moodle > /dev/null 2>&1 &
    - moodle-plugin-ci phpunit
rules:
    - exists:
    - $CI_PROJECT_DIR/moodle/local/plugins/tests/*_test.php

Behat features:
stage: test
script:
    - cd moodle
    - php admin/tool/behat/cli/init.php
    - moodle-plugin-ci behat --suite default --profile chrome
rules:
    - exists:
    - $CI_PROJECT_DIR/moodle/local/plugins/tests/behat/*.feature
```

## Conclusion

By implementing an automated testing pipeline using GitLab CI/CD and moodle-plugin-ci, Moodle plugin developers can ensure their plugins meet the required standards and maintain compatibility with Moodle versions. The provided example in the gitlab-ci.yml file demonstrates the various stages and tests that can be included in the pipeline. With continuous integration and automated testing, developers can deliver high-quality Moodle plugins, contributing to the Moodle community's growth and success.

Remember to customize and adapt the pipeline to suit your specific plugin development needs. Happy coding and testing!
