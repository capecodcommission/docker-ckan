# Docker Compose setup for CKAN

* [Overview](#overview)
* [Quick start](#quick-start)
* [Development mode](#development-mode)
   * [Create an extension](#create-an-extension)
   * [Running the debugger (pdb / ipdb)](#running-the-debugger-pdb--ipdb)
* [CKAN images](#ckan-images)
   * [Extending the base images](#extending-the-base-images)
   * [Applying patches](#applying-patches)
* [Known Issues](#known-issues)


## Overview

This is a set of Docker images and configuration files to run a CKAN site

It is largely based on two existing projects:

* Keitaro's [CKAN Docker images](https://github.com/keitaroinc/docker-ckan)
* Docker Compose setup currently included in [CKAN core](https://github.com/ckan/ckan)


It includes the following images, all based on [Alpine Linux](https://alpinelinux.org/):

* CKAN: modified from keitaro/ckan (see [CKAN Images](#ckan-images)) for more details)
* DataPusher: modified from keitaro/datapusher
* PostgreSQL: Official PostgreSQL image
* Solr: official Solr image with CKAN's schema
* Redis: standard Redis image

The site is configured via env vars (the base CKAN image loads [ckanext-envvars](https://github.com/okfn/ckanext-envvars)), that you can set in the `.env` file.

## Quick start

Copy the included `.env.example` and rename it to `.env` to modify it depending on your own needs.

Using the default values on the `.env.example` file will get you a working CKAN instance. There is a sysadmin user created by default with the values defined in `CKAN_SYSADMIN_NAME` and `CKAN_SYSADMIN_PASSWORD`(`ckan_admin` and `test1234` by default). I shouldn't be telling you this but obviously don't run any public CKAN instance with the default settings.

To build the images:

	docker-compose build

To start the containers:

	docker-compose up

## CCC

### Docker dev flow

1. make a spot to hold your cloned project
2. cd to above location and `git clone https://github.com/capecodcommission/docker-ckan.git`
3. `cd docker-ckan`
4. `cp .env.example .env`
5. Follow instructions above to build images and then start the containers
6. When you want to stop images, press CTRL + "c" key
7. `docker-compose down`
8. If no changes have been made to the code base, `docker-compose up`
9. If changes have been made, re-build images `docker-compose build` and start the containers `docker-compose up`

## Git flow

1. ``git status`` -- this should return that you're on your feature branch (feature-working-on/INITIALS), if not ``git checkout feature-working-on/INITIALS)``
2. ``git checkout dev``
3. ``git pull origin dev`` -- this is essential and provides the up-to-date code base from the repository. if someone else has pushed changes since you last have pulled, you will now have those before updating.
4. ``git checkout feature-working-on/INITIALS``
5. Assure that the services run in your feature branch by stepping through 'Getting app running in docker containers'
6. Alert others on the project that you're merging your feature into dev so as not to merge at the same time to avoid merge conflicts
7. ``git rebase -i dev``
8. If you have multiple commits, in the text editor that comes up, replace the letters "p" or words "pick" with "s" for "sqaush" next to the commits all the commits underneath the top ("oldest" or 1st commit in your sets of commit).
9. If there are any conflicts, go through those: ``git add``, ``git rebase --continue`` until complete
10. Once the terminal returns a successful rebase, ``git checkout dev``
11. ``git checkout dev``
12. ``git merge feature-working-on/INITIALS``
13. Assure that the services run in your dev branch by stepping through 'Getting app running in docker containers'
14. ``git status`` -- should return that you're 1 commit ahead of origin/dev
15. ``git push origin dev``
16. ``git status`` -- should return that you're on dev branch and that it's 'already up to date.' and pointing at repo dev branch
17. ``git branch -D feature-working-on/INITIALS`` -- delete your local feature branch that you just merged into dev
18. ``git checkout feature-working-on/INITIALS`` -- start over

## Resources
Create a sysadmin: https://docs.ckan.org/en/2.8/maintaining/getting-started.html#create-admin-user
Customize look & feel w/ sysadmin: https://docs.ckan.org/en/2.8/sysadmin-guide.html#customizing-look-and-feel
Docker-compose documentation: https://docs.ckan.org/en/2.8/maintaining/installing/install-from-docker-compose.html
Template customization: https://docs.ckan.org/en/2.8/theming/templates.html?highlight=customizing

## Development mode

To develop local extensions use the `docker-compose.dev.yml` file:

To build the images:

	docker-compose -f docker-compose.dev.yml build

To start the containers:

	docker-compose -f docker-compose.dev.yml up

See [CKAN Images](#ckan-images) for more details of what happens when using development mode.


### Create an extension

You can use the paster template in much the same way as a source install, only executing the command inside the CKAN container and setting the mounted `src/` folder as output:

    docker-compose -f docker-compose.dev.yml exec ckan-dev /bin/bash -c "paster --plugin=ckan create -t ckanext ckanext-myext -o /srv/app/src_extensions"

The new extension will be created in the `src/` folder. You might need to change the owner of its folder to have the appropiate permissions.


### Running the debugger (pdb / ipdb)

To run a container and be able to add a breakpoint with `pdb` or `ipdb`, run the `ckan-dev` container with the `--service-ports` option:

    docker-compose -f docker-compose.dev.yml run --service-ports ckan-dev

This will start a new container, displaying the standard output in your terminal. If you add a breakpoint in a source file in the `src` folder (`import pdb; pdb.set_trace()`) you will be able to inspect it in this terminal next time the code is executed.


## CKAN images

```
    +-------------------------+                +----------+
    |                         |                |          |
    | openknowledge/ckan-base +---------------->   ckan   | (production)
    |                         |                |          |
    +-----------+-------------+                +----------+
                |
                |
    +-----------v------------+                 +----------+
    |                        |                 |          |
    | openknowledge/ckan-dev +----------------->   ckan   | (development)
    |                        |                 |          |
    +------------------------+                 +----------+


```

The Docker images used to build your CKAN project are located in the `ckan/` folder. There are two Docker files:

* `Dockerfile`: this is based on `openknowledge/ckan-base` (with the `Dockerfile` on the `/ckan-base/<version>` folder), an image with CKAN with all its dependencies, properly configured and running on [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/) (production setup)
* `Dockerfile.dev`: this is based on `openknowledge/ckan-dev` (with the `Dockerfile` on the `/ckan-dev/<version>` folder), wich extends `openknowledge/ckan-base` to include:

  * Any extension cloned on the `src` folder will be installed in the CKAN container when booting up Docker Compose (`docker-compose up`). This includes installing any requirements listed in a `requirements.txt` (or `pip-requirements.txt`) file and running `python setup.py develop`.
  * The CKAN image used will development requirements needed to run the tests .
  * CKAN will be started running on the paster development server, with the `--reload` option to watch changes in the extension files.
  * Make sure to add the local plugins to the `CKAN__PLUGINS` env var in the `.env` file.

From these two base images you can build your own customized image tailored to your project, installing any extensions and extra requirements needed.

### Extending the base images

To perform extra initialization steps you can add scripts to your custom images and copy them to the `/docker-entrypoint.d` folder (The folder should be created for you when you build the image). Any `*.sh` and `*.py` file in that folder will be executed just after the main initialization script ([`prerun.py`](https://github.com/okfn/docker-ckan/blob/master/ckan-base/setup/prerun.py)) is executed and just before the web server and supervisor processes are started.

For instance, consider the following custom image:

```
ckan
├── docker-entrypoint.d
│   └── setup_validation.sh
├── Dockerfile
└── Dockerfile.dev

```

We want to install an extension like [ckanext-validation](https://github.com/frictionlessdata/ckanext-validation) that needs to create database tables on startup time. We create a `setup_validation.sh` script in a `docker-entrypoint.d` folder with the necessary commands:

```bash
#!/bin/bash

# Create DB tables if not there
paster --plugin=ckanext-validation validation init-db -c $CKAN_INI
```

And then in our `Dockerfile` we install the extension and copy the initialization scripts:

```Dockerfile
FROM openknowledge/ckan-dev:2.7

RUN pip install -e git+https://github.com/frictionlessdata/ckanext-validation.git#egg=ckanext-validation && \
    pip install -r https://raw.githubusercontent.com/frictionlessdata/ckanext-validation/master/requirements.txt

COPY docker-entrypoint.d/* /docker-entrypoint.d/
```

### Applying patches

When building your project specific CKAN images (the ones defined in the `ckan/` folder), you can apply patches 
to CKAN core or any of the built extensions. To do so create a folder inside `ckan/patches` with the name of the
package to patch (ie `ckan` or `ckanext-??`). Inside you can place patch files that will be applied when building
the images. The patches will be applied in alphabetical order, so you can prefix them sequentially if necessary.

For instance, check the following example image folder:

```
ckan
├── patches
│   ├── ckan
│   │   ├── 01_datasets_per_page.patch
│   │   ├── 02_groups_per_page.patch
│   │   ├── 03_or_filters.patch
│   └── ckanext-harvest
│       └── 01_resubmit_objects.patch
├── Dockerfile
└── Dockerfile.dev

```


## Known Issues

* Running the tests: Running the tests for CKAN or an extension inside the container will delete your current database. We need to patch CKAN core in our image to work around that.
