---
layout: post
title: GitLab CI and Docker at CERN (and probably other places)
---
![Aerial bow view of the OBO carrier 'Naess Crusader' on sea trials, June 1973](/resources/shipping_containers.jpg)

_This entry was intended to be cross-posted to
[the CERN Databases blog](http://db-blog.web.cern.ch/), but is currently
pending review. Consider it a pre-release version_.

CERN's IT department uses GitLab for internal projects, which attempts
to cover the full source management (GitHub), continuous integration
(Travis), and Docker workflow.

## Setting up a CI pipeline

Much like Travis on GitHub, GitLab's continuous integration is
configured via a YAML file in the project root directory, called
`.gitlab-ci.yml`. It's got its
[own documentation](https://docs.gitlab.com/ce/ci/README.html),
fortunately.

CI jobs can be pipelined, that is made to run in succession. A good
practice is probably to first run low-level build tasks and/or linting,
then unit tests, and then (if they pass) any deployment scripts, docker
image builds et cetera. As CERN's GitLab instance is self-hosted, it
also has access to a number of more advanced runners which may--for
example--have shared file systems such as AFS or DFS, or be able to
perform other specific tasks. Windows runners are also theoretically
possible, as I understand it.

FIXME: Picture of the deploy pipeline for cern-sso-python

Here is a sample `.gitlab-ci.yml`:

```yaml
lint:
  image: python:latest
  stage: build
  script:
    - pip install -r ci-deps.txt
    - flake8 cern_sso.py

test_2:
  image: python:2
  script:
    - pip install -r requirements.txt
    - pip install -r ci-deps.txt
    - pytest

test_3:
  image: python:3
  script:
    - pip install -r requirements.txt
    - pip install -r ci-deps.txt
    - pytest

rpm:
  image: ubuntu:latest
  stage: deploy
  only:
    - tags

  artifacts:
    paths:
      - dist/*.rpm
    name: "dist-$CI_BUILD_REF_NAME"

  script:
    - apt-get update && apt-get install -y rpm libxslt-dev libxml2-dev python-pip libkrb5-dev
    - pip install twine
    - python setup.py bdist --format=rpm
```

As you can see, this setup first runs a linting step ("build"), then
executes the tasks `test_2` and `test_3`, which runs unit tests in
Python 2 and 3 respectively and in parallel, and builds an RPM if all
the previous steps succeeded, and a tagged release was
pushed. E.g. pushing the tag "1.0.0" will build and install a
corresponding RPM. GitLab's key `artifacts` is used to save the RPM in
the repository, but other methods could be used.

Different Docker images are being used as bases for the various steps:
smaller Python images (version 2 and 3 respectively) are being used for
testing, while a full Ubuntu image is used for RPM generation (which may
or may not be a waste of resources).

## Pushing Docker images to GitLab's private registry

The
[official instructions](https://docs.gitlab.com/ce/ci/docker/using_docker_build.html)
for using Docker-in-Docker to run Docker commands on the CI runners does
not work at CERN, possibly due to security concerns. In stead, a
specific job tag is provided which runs jobs on a separate runner,
which _only_ builds and uploads Docker images. It is used like this:

```yaml
docker-image:
  stage: deploy
  only:
    - tags

  tags:
    - docker-image-build

  variables:
    TO: $CI_REGISTRY_IMAGE:$CI_BUILD_TAG

  script: ""
```

The variable `$CI_REGISTRY_IMAGE` corresponds to something like
`gitlab-registry.cern.ch/user-or-group/my-repository`. The following
variable is the image tag (here using the provided Git tag, analogous to
the example with the RPM above), if omitted you will push to
`:latest`. `script` is left empty as you cannot provide any instructions
here in any case.

Please note that _you need to manually enable the Docker Registry_ for
your repository for this to work. Do this via the cog wheel menu, "Edit
Project" and checking "Container Registry" under "Features".

Finally, it is worth noting that one can use internal Docker images from
the GitLab registry as bases for runners, though. More on that in the
section on pushing documents to web pages below!

See also:
- [KB0003690](https://cern.service-now.com/service-portal/article.do?n=KB0003690): How to use gitlab-ci for Continuous Integration of GitLab projects
- [KB0004284](https://cern.service-now.com/service-portal/article.do?n=KB0004284): Getting started with the GitLab container Registry
- [`build_docker_image` example project](https://gitlab.cern.ch/gitlabci-examples/build_docker_image) (internal)

## Managing private dependencies in Python projects

Python's `pip` can install software directly from Git repositories,
including private ones--as long as you have the credentials to access
them. Basically, I see two approaches: either you use a standard
`requirements.txt` file, which may look a bit like this:
```
-e git+ssh://git@gitlab.cern.ch:7999/astjerna/netapp-ocum-events.git#egg=netapp-ocum-events
flake8
sqlalchemy==1.1.0b3
MySQL-python==1.2.5
...
```

Or you can have an early `build` step in your deploy pipeline which
clones the necessary repositories to--for example--a `lib/` folder in
your project. In that case, your `.gitlab-ci.yml` could look like this:

```yaml
deps:
  image: python:2
  stage: prebuild

  before_script:
  - 'which ssh-agent || ( apt-get update -y && apt-get install openssh-client -y )'
  - eval $(ssh-agent -s)
  - ssh-add <(echo "$SSH_PRIVATE_KEY")
  - mkdir -p ~/.ssh
  - '[[ -f /.dockerenv ]] && echo -e "Host *\n\tStrictHostKeyChecking no\n\n" > ~/.ssh/config'

  artifacts:
    paths:
      - lib

  script:
    - make deps
```

Take special note of the `artifacts` part, which makes sure the
resulting directory is passed on to the following pipeline steps.

If using this approach, your `requirements.txt` could look like this:

```
...
sphinx==1.4.8
recommonmark==0.4.0
sphinx-rtd-theme==0.1.9

lib/cern-sso-python
lib/netapp-ocum-events
```

Finally, for reference, the `make deps` rule looks like this:

```
deps:	## Build and install all requirements, including private dependencies stored in ./lib/
	rm -rf lib/* && mkdir -p lib && cd lib && \
	git clone ssh://git@gitlab.cern.ch:7999/db/cern-sso-python.git && \
	git clone ssh://git@gitlab.cern.ch:7999/astjerna/netapp-ocum-events.git
	pip install -r requirements.txt

.PHONY: deps
```

Authentication is normally done--for both cases--using
[Deploy Keys, as per normal instructions](https://docs.gitlab.com/ce/ci/ssh_keys/README.html).

A special note on using private repositories in `setup.py` scripts. You
can provide them in `dependency_links` like this:

``` python
    dependency_links=[("git+ssh://git@gitlab.cern.ch:7999/"
                       "astjerna/netapp-ocum-events.git"
                       "#egg=netapp-ocum-events-0.4.1"),
                      ("git+ssh://git@gitlab.cern.ch:7999/"
                       "db/cern-sso-python.git"
                       "@1.0.3#egg=cern-sso-1.0.3")],
```

But in order for that to work when installing your project using `pip`
you need to add an extra argument `--process-dependency-links`:

```
$ pip install --process-dependency-links "git+ssh://git@gitlab.cern.ch:7999/astjerna/netapp-event-syncd.git"
```

## CERN-specific tricks

### Kerberos in CI jobs

### Running Docker Images

### Deploying documentation to web pages with DFS

### Fetching images from GitLab's registry in Puppet using Teigi


_Image from [Tyne & Wear Archives & Museums](https://www.flickr.com/photos/twm_news/25316294942/)_.
