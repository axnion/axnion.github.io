---
layout: post
title:  "TravisCI with multi-language repository"
date:   2017-08-04 07:27:00 +0200
---
## Background
Last semester I was in a group project in a university course where we where assigned a customer who we designed and implemented a piece of software for. We ended up with a system using Java 8 and Spring for the back end, and a React client for the front end. You can find the project [here](https://github.com/2DV603NordVisaProject/nordvisa_calendar).

But me being me felt like having a bit of fun after spending hours upon hours writing Java code for this project. I decided to implement my usual combination of TravisCI and CodeCov to test the software on a central system, and display test coverage to "motivate" the group for better test coverage. I quickly realized a problem (other than the test coverage :P).

## The Problem
The structure of our project repository did not work well with how I've normally configured TravisCI. Both server and client is in separate directories in the same repository. This to us felt like a sane way to structure our project. 

But let's looks at how TravisCI is configured. The configuration is kept in the `.travis.yml` file in the root of the repository. When TravisCI is triggered to build it will use this .yml file to create the build environment for that repository. Possibly the most important setting in the configuration is the `language` variable which is where you set what type of language or platform you want your build environment to have. If the project is using Java I can set `language: java` and recive a environment for testing and building Java applications, or if it's Ruby I can define it as `language: ruby` and so on. But it can only be defined once in the configuration.

So if I can only define one environment, how do we then test or build a project from a repository using multiple languages or platforms? I my case I needed both Java and NodeJS to be installed to test both client and server.

## Solution
My first thought was to just use the predefined Java configuration and then in the before_script part install NodeJS manually, it is a normal ubuntu machine after all. This would most likely have worked fine, but it felt like a very clumsy solution and a horrible hack. Surely TravisCI had thought about this scenario.

I instead found the [Matrix](https://docs.travis-ci.com/user/customizing-the-build/#Build-Matrix) feature. This feature lets you hav several builds in a single configuration, so kind of like having several `.travis.yml` configs executing in parallel. But if we look in the documentation it's generally talked about as a feature to test and build the software on different operating systems or different versions of the programming language, but of course it can also be used to test different sub-projects within a repository using different languages.

Below is a simple example on how you can use matrix to have separate builds for each language in one `.travis.yml` file. You simply add the `matrix:` and `include:` to the configuration and then each element in the array is a build. For each "component" of your project you can just add another element in the array and configure the build as you would normally do in the `.travis.yml`.
{% highlight ruby %}
matrix:
  include:
    - language: java
      jdk: oraclejdk8
      script:
        - gradle check
    - language: node_js
      script:
        - npm test
{% endhighlight %}


Here is the full `.travis.yml` used in the project I discussed in the background.
{% highlight ruby %}
dist: trusty
sudo: required
matrix:
  include:
    - language: java
      jdk: oraclejdk8
      before_script:
        - cd server
      script:
        - gradle check --info
        - gradle jacocoTestReport
      before_cache:
        - rm -f  $HOME/.gradle/caches/modules-2/modules-2.lock
        - rm -fr $HOME/.gradle/caches/*/plugin-resolution/
      cache:
        directories:
          - $HOME/.gradle/caches/
          - $HOME/.gradle/wrapper/
      after_success:
        bash <(curl -s https://codecov.io/bash) -cF server
    - language: node_js
      node_js:
        - "node"
      before_script:
        - cd client
        - npm install
        - npm install istanbul-lib-instrument
      cache:
        directories:
          - "node_modules"
      script:
        - npm test
        - npm run coverage
      after_success:
        bash <(curl -s https://codecov.io/bash) -cF client

notifications:
    email: false
{% endhighlight %}

I hope this helps someone, since I had a hard time finding any information on this very specific problem.

// Axel Nilsson