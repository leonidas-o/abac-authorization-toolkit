# abac-authorization-toolkit
This is an ansible based toolkit to help to incorporate the abac-authorization package into a Swift Vapor project.

## Introduction
Default variables will be set in `group_vars/all/0_vars.yml` and in the ansible roles e.g. `sample_frontend_backend/defaults/main.yml`.
If you want to overwrite some of the variables from `group_vars` copy over the variables into `group_vars/all/1_vars.yml` and modify the values according to your needs. For a role like `sample_frontend_backend` copy over the role variables into `sample_frontend_backend/vars/main.yml` and modify the values according to your needs. Any ansible role variable can be overwritten in its `/vars/main.yml` file.
Dictionaries must be copied over as a whole, single key values can't be overwritten with the current config (It's not recommended to change `hash_behaviour` settings).
Single parts can be removed from the sample generation process by setting `is_active` to false.

## Getting Started
1. Create new vapor project
```bash
vapor new hello -n
```
> If using static files make sure you've set the `custom working directory` in e.g. XCode.
1. Clone the abac-authorization-toolkit or pull the latest changes and cd into the abac-authorization-toolkit project
2. Set `VAPORPROJECT` var, run the playbook and watch your project grow
```bash
VAPORPROJECT=/path/to/vapor-project
docker container rm abac-toolkit || true && docker run \
    --name abac-toolkit -it \
    -v $PWD:/srv/ansible:z \
    -v $VAPORPROJECT:/srv/vapor:z \
    --workdir /srv/ansible cytopia/ansible:2.13-tools \
    /bin/bash -c "ansible-playbook site.yml --tags sample-frontend-backend"
```
> RedisRepo is the only template right now, so e.g. fluent for sessions is not supported (yet).

Possible Tags:
- `--tags sample-frontend-backend`: Pre-configured frontend/backend project with Fluent, Redis, Leaf and PostgreSQL
- ~~`--tags sample-api`: Not yet implemented~~
- ~~`--tags sample-backend`: Not yet implemented~~
> PostgreSQL can be replaced easily by modifying `configure_block` and `swift_packages` vars OR by a PR with a similar ansible role but another db driver.


If using an ansible role with a DB (e.g. PostgreSQL) make sure you've created the db and user according the `configure_block` in the roles `defaults/main.yml` or if you have overwritten it, in `vars/main.yml`.
After the ansible playbook has finished, the project should compile successfully and be ready to start.



## Troubleshooting
List all tasks for the specified tags (won't be executed, only listed)
```bash
VAPORPROJECT=/path/to/vapor-project
docker container rm abac-toolkit || true && docker run \
    --name abac-toolkit -it \
    -v $PWD:/srv/ansible:z \
    -v $VAPORPROJECT:/srv/vapor:z \
    --workdir /srv/ansible cytopia/ansible:2.13-tools \
    /bin/bash -c "ansible-playbook site.yml --tags sample-frontend-backend --list-tasks"
```
> If ansible thinks that file already updated: `--flush-cache`


## Customization
Ansible config settings are set in `ansible.cfg`. All settings can be listed with
```bash
docker container rm abac-toolkit || true && docker run \
    --name abac-toolkit -it \
    -v $PWD:/srv/ansible:z \
    --workdir /srv/ansible cytopia/ansible:2.13-tools \
    /bin/bash -c "ansible-config init --disabled -t all"
```
