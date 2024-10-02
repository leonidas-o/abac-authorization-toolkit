# abac-authorization-toolkit
This is a toolkit to help to incorporate the abac-authorization package into a Swift Vapor project.

## Getting Started
Default variables will be set under:
`group_vars/all/0_vars.yml` and in a role e.g. `sample_frontend_backend/defaults/main.yml`.
If you need to overwrite some of the variables, For `group_vars`: copy over the variable and modify the values according to your needs inside `group_vars/all/1_vars.yml`. For a role like `sample_frontend_backend`: copy over the variable, create the directory and modify the values according to your needs inside `sample_frontend_backend/vars/main.yml`. Any ansible role vars can be overwritten in the `/vars/main.yml` file.
Dictionaries must be copied over as a whole, single key values can't be overwritten with the current configs (It's not recommended to change `hash_behaviour` settings).
Single parts can be removed from the sample generation process by setting `is_active` to false.


1. Create new vapor project
```bash
vapor new hello -n
```
> If using static files make sure you've set the `custom working directory` in e.g. XCode.
2. The latest abac-authorization-toolkit changes were pulled and current directory is root of the project
3. Set `VAPORPROJECT` var, run the ansible playbook and watch your project grow
```bash
VAPORPROJECT=/path/to/vapor-project
docker container rm abac-toolkit || true && docker run \
    --name abac-toolkit -it \
    -v $PWD:/srv/ansible:z \
    -v $VAPORPROJECT:/srv/vapor:z \
    --workdir /srv/ansible cytopia/ansible:2.13-tools \
    /bin/bash -c "ansible-playbook site.yml --tags sample-frontend-backend"
```
> It comes pre-configured to use redis for sessions. RedisRepo is the only template right now, so e.g. fluent for sessions is not supported (yet).

Possible Tags:
`--tags sample-frontend-backend`: Pre-configured frontend/backend project with Fluent, Redis, Leaf and PostgreSQL
~~`--tags sample-api`: Not yet implemented~~
~~`--tags sample-backend`: Not yet implemented~~
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
