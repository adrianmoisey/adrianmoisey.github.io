---
layout:     post
title:      "Parallel jobs in GitLab-Runner with Pytest"
date:       2018-11-27 17:00:00
author:     "Adrian Moisey"
---

# Parallel jobs in GitLab with Pytest

On 22 November 2018, GitLab released version [11.5](https://about.gitlab.com/2018/11/22/gitlab-11-5-released/) which introduced a new attribute for the GitLab Runner called `parallel`.  
This option allows you to split a single job into multiple jobs, and let them run in parallel.  
I'm going to describe how you can use this feature to split up tests into multiple jobs with `pytest`.

## Configuring your job

The config option is quite simple, you define the number of parallel jobs to run using `parallel: N` in your job definition. The [GitLab Runner](https://docs.gitlab.com/ee/ci/yaml/#parallel) docs have more details.
Here's an example from a snippet of my `.gitlab-ci.yml` file:
```
pytest:
  stage: test
  image: $CONTAINER_IMAGE:latest
  parallel: 5
  services:
    - mysql:5.5
    - redis
  script:
    - pytest
```

This will run 5 instances of `pytest`, with two additional environment variables. `CI_NODE_TOTAL` and `CI_NODE_INDEX`.  
`CI_NODE_TOTAL` will give you the number of jobs that are run, while `CI_NODE_INDEX` will be the number of the instance. For example, the third job will be `CI_NODE_TOTAL=5` and `CI_NODE_INDEX=3`.

## Configuring pytest

Standard `pytest` doesn't support splitting tests between multiple jobs, but, fortunately for us, there is a plugin to do that.  
[pytest-test-groups](https://github.com/mark-adams/pytest-test-groups) allows you to split tests up into groups, by passing in a count, and the index of that run. After installing pytest-test-groups, we can tweak the `.gitlab-ci.yml` file like this:

```
pytest:
  stage: test
  image: $CONTAINER_IMAGE:latest
  parallel: 5
  services:
    - mysql:5.5
    - redis
  script:
    - pytest --test-group-count $CI_NODE_TOTAL --test-group=$CI_NODE_INDEX
```

Now, when we run the job, we should get output similar to this in the "pipeline" section of GitLab:
<img src="{{ site.baseurl }}/img/parallel.png" alt="GitLab-Runner Parallel">

## Final

As you can see, this feature is quite simple to use and configure. Some things to note before considering it:
1. The idea of running jobs in parallel is to make your tests run faster. But depending on your setup, sometimes this isn't the case. On the code base I tested, our database migrations were slow, which caused our tests to run slower in parallel. Considering this is an easy setup, it's worth testing.
1. If you use tools that generate reports (such as code coverage) you'll end up with multiple reports. This means you may need to figure out how to combine these reports to make them useful again.
