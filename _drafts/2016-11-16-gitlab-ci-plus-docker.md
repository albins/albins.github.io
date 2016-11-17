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
also has access to a number of more advanced runners which may---for
example---have shared file systems such as AFS or DFS, or be able to
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
including private ones---as long as you have the credentials to access
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
clones the necessary repositories to---for example---a `lib/` folder in
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

Authentication is normally done---for both cases---using
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

If your jobs require Kerberos credentials, you can inject for example
keytabs as you would any other secret, such as SSH keys and
similar. However, Kerberos keytabs are binary, but you can use base64 to
serialise them.

Make a keytab for the service account as per the instructions in
[KB0002449](https://cern.service-now.com/service-portal/article.do?n=KB0002449)
(in this instance, `dbstoragemon` is the user name).

Then, add your keytab as a variable to the repository:

```
$ base64 --wrap=0 < dbstoragemon.keytab | xclip -sel clip
```

Create a new variable in GitLab named `KRB_KEYTAB_CONTENTS` and paste
the keytab into the data field. The command above assumes that you are
on GNU/Linux and have `xclip` installed. You could just as well remove the
pipe and everything after it and hand-copy the base64-encoded string
from the terminal output.

Add the following to your `.gitlab-ci.yml`:

``` yaml

before_script:
    # Decode the base64-encoded keytab and store it in dbstoragemon.keytab.
    - base64 --decode <(echo "$KRB_KEYTAB_CONTENTS") > dbstoragemon.keytab
    # Authenticate using the keytab:
    - kinit dbstoragemon@CERN.CH -k -t dbstoragemon.keytab
```

And _voilá_, you have a valid Kerberos ticket in your runner (provided
that you have installed a working Kerberos environment. Here is an
example of a full CI job running on a standard CERN image (some
non-interesting details have been removed):

``` yaml
install_run:
  image: cern/cc7-base
  before_script:
    - yum -y update
    - yum install -y krb5-workstation krb5-libs
    # Decode the base64-encoded keytab and store it in dbstoragemon.keytab.
    - base64 --decode <(echo "$KRB_KEYTAB_CONTENTS") > $KRB_USERNAME.keytab

  script:
    - kinit $KRB_USERNAME@CERN.CH -k -t $KRB_USERNAME.keytab
    - python2.7 setup.py install
    - cern-get-sso-cookie.py --help
    - cern-get-sso-cookie.py --kerberos --verbose --url https://dbnas-storage-docs.web.cern.ch
    - grep -q "dbnas-storage-docs.web.cern.ch" cookies.txt
```

As you can se, the install script installs a Python program on a
standard CERN CentOS installation, makes sure the command-line program
is available and produces its help output, and tries to execute and
validate a command---in this instance, authenticating against a protected
resource behind CERN's Single Sign-On portal---verifying that it did
indeed get a valid cookie for the resource.

### Deploying documentation to web pages with DFS

A typical thing one might want to do is to generate documentation and
upload it somewhere. You might be using Sphinx or GitBook for
this. Thanks to a
[custom-made web deployer Docker image produced by Borja Aparicio Cotarelo](https://gitlab.cern.ch/ci-tools/ci-web-deployer/),
this is fairly painless. Documentation needs to be built in a separate
build step, and then passed as artifacts to the Wed Deployer. A subset
of the build pipeline may look like this:

``` yaml
doc:
  stage: test
  image: python:2

  before_script:
    - pip install -r requirements.txt

  script:
    - mkdir -p public/netapp-syncd
    - cd doc && make html
    - cp -r _build/html/* ../public/netapp-syncd
  artifacts:
    paths:
      - public

dfsdeploy:
  stage: deploy
  only:
    - tags
  image: gitlab-registry.cern.ch/ci-tools/ci-web-deployer:latest
  script:
    - deploy-dfs
```

For this to work, a few variables need to be set to provide the access
credentials for the web page you are uploading to. See the CI Web
deployer README for more details!

### Fetching images from GitLab's registry in Puppet using Teigi

There
[is a Puppet module for Docker](https://github.com/garethr/garethr-docker). However,
CERN stores secrets in
[a system called Teigi](https://twiki.cern.ch/twiki/bin/view/Main/TeigiVault),
which doesn't seem to be used anywhere else. And that system, by design,
makes injecting secrets as variables the way the Docker module expects
them slightly complicated. As a work-around, we can just use the normal
shell UI and create a simple single-line script for performing the
actual authentication:

``` puppet
$docker_auth_script = '/etc/docker-auth.sh'

teigi::secret::sub_file{$docker_auth_script:
  teigi_keys => [gitlab_password, gitlab_user],
  template   => 'hg_playground/astjerna-docker-auth.sh.erb'
}

exec{'docker_auth':
  command   => "/usr/bin/bash ${docker_auth_script}",
  subscribe => File[$docker_auth_script]
}
```

That template, in turn, is just a simple fill-in-the-blanks one-liner:

``` erb
/usr/bin/docker login -u  %TEIGI__gitlab_user__% -p  %TEIGI__gitlab_password__% -- gitlab-registry.cern.ch
```

Finally, once you have that script, you can add it as a dependency to
the normal docker image instance:

``` puppet
$registry_path = 'gitlab-registry.cern.ch/astjerna'

docker::image {"${registry_path}/netapp-event-syncd":
  image_tag => $image_version,
  require   => Exec['docker_auth']
}
```

Which will fetch the image from the private registry on GitLab.

_Image from [Tyne & Wear Archives & Museums](https://www.flickr.com/photos/twm_news/25316294942/)_.
