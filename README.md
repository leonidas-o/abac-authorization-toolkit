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




## Tag Descriptions

### "sample-frontend-backend"

This demo-project (brought together under one hood out of two projects - api and backend) shows how a separate API and backend approach could work. Detailed instructions how to use ABACAuthorization package can be found in the package's README file. Http Port can be set via HTTP_PORT Environment variable, if not set, it defaults to 8080.

> IMPORTANT: At the beginning, the most routes will throw a 403: Forbidden. This is expected, as the default Auth Policies are only providing minimal access (see Auth Policies section).

#### Login
A default admin user will be created on first start.

Username: `webmaster@foo.com` and a random generated password (shown only once in the console).

> CAUTION: The password will be generated and shown only on a first migration/ seeding of the admin user. See `AdminUserMigration.swift` file in User -> Models for more Info.


#### Background Info
`ABACMiddleware` is used for securing the API.
For backend services where sessions are used, you need e.g. a `UserAuthSessionsMiddleware`. It's basically a simple AuthSessionsMiddleware + RedirectMiddleware with a little bit of logic for access tokens. 
That means if you access the api directly via a rest, iOS etc. clients, you don't need a `UserAuthSessionsMiddleware`. You also don't need the policy creation GUI etc. if you examine the `ABACAuthorizationPolicyController` you will find a bulk api endpoint which you can use, OR you can feed the database directly OR build your own policy creation GUI etc.
Models with the `...Model` suffix are pure Vapor models and reference types. There can be Data-Transfer-Objects, these are structs, value types, and are used for transferring data between the API and clients. This DTO's can be shared across projects (e.g. Vapor API and iOS client) using a separate swift package.
For example here `UserModel` and `User` or `RoleModel` and `Role`.

A big benefit of the ABACAuthorizationPolicy, you don't need to restart the api/service after adding or modifying policies. It's runtime configurable.


#### Auth Policies
1. Take a look at the default `Auth Policies`. Only the policies you see in here, allows actions on the resource. Everything else will throw a 403: Forbidden error.
2. Therefore a `read auth policies` and `read roles` will be created by default. If you want to be able to `read` - `Todo's` or `Users` simply create the specific policy. 
3. You can't delete or update a policy? Do you have a policy which allows you to do that? 


#### ABACConditions
Conditions (Attributes) are not mandatory and can be neglected if not needed. If needed they can be created on all "cached" values. That means, everything in `AccessData.UserData` can be used.

Starting from within `UserData` Model, specify a path using dot notation. 
Condition examples: 
- `user.name`, 
- `roles.0.name`

So you could build a policy with a condition like: 
- Operation on type `string`
- Operation itself `==`
- Left hand side is a `reference` to `user.name`
- Right hand side is a `value` for example `foo`

Pretty straight forward, that would only grant access if the users name is equal to `foo`.
> The key has to be unique for that specific authorization policy e.g. key1, key2, ... .


#### Advice
See `UserController` routes -> Internal vs External routes. All the internal routes make use of ABACAuthorization where you need such a system. External routes like all "MyUser" routes, don't need such an authorization as a user is allowed to change his own data.



#### Horizontal scaling

> This approach is mostly made with Kubernetes' headless service in mind, you can modify your  `_recreateAllInMemoryPolicies` route handler to fit your needs. For example injecting an array with all api instances etc.

To achieve a fast decision making process for the evaluation if a request should be permitted or denied, all ABAC policies are stored in memory. This approach leads to some extra work to keep all instances, their in-memory policies, in sync.   
See the `ABACAuthorizationPolicyController`, there is a route handler called `_recreateAllInMemoryPolicies`, which should be requested with an `address` url query, pointing to the headless service.
Without the `address` url query an api instance will simply recreate all its in memory policies with the data from the configured database.
In a dynamic environment like a Kubernetes cluster, where your api instances can be re-created and deployed on different nodes at any time (IP addresses can change), you need a mechanism to address each api instance. Therefore an `nslookup` on a Kubernetes headless service, gives you all ip addresses of your currently running api instances.
To execute such a lookup using swift, we make us of OpenKitten's DNSClient (https://github.com/OpenKitten/NioDNS.git). Afterwards, the route controller requests itself recursively while iterating over the array with api addresses but the request for each instances `_recreateAllInMemoryPolicies` route does NOT contain an `address` url query, so each api instance recreates its in-memory policies.
As this is a protected route, the SystemBot user has to be logged in and use its token to send the update request to all instances.




## License

This project is licensed under the MIT License - see the LICENSE.md file for details.
