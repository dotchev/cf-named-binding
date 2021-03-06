# Named Service Bindings
Proposal for named service bindings in Cloud Foundry

## Motivation

An application in Cloud Foundry can be [bound][1] to multiple service instances.
Also one service instance can be bound to multiple applications, so the relation is many-to-many.

Selecting the right service instance inside the application is error prone as it involves some kind of guessing.

Here is an example of environment variable [VCAP_SERVICES] which provides an application with information about bound service instances:
```
VCAP_SERVICES=
{
  "elephantsql": [
    {
      "name": "elephantsql-c6c60",
      "label": "elephantsql",
      "tags": [
        "postgres",
        "postgresql",
        "relational"
      ],
      "plan": "turtle",
      "credentials": {
        ...
      }
    },
    ...
  ],
  ...
```
Note that:
* Service instance properties are unknown in advance, when the application is developed. These are known only at time of deployment. Also they may vary across different deployment environments. So it is not safe to hard-code their values in the application.
* `name` is the only unique service instance property, which is determined when the service instance is created

There are situations when an applications uses different instances of the same service for different purposes. Then it is particularly hard to select the right instance. For example an application might use two instances of a database service - one for persistance of regular objects and the other for sensitive data like user credentials.

An application should not require the operator to use specific instance names as one service instance can be bound to multiple applications.

One solution is to use additional environment variables to tell the application the purpose of each service instance, e.g.
```
env:
  REGULAR_STORE: elephantsql-c6c60
  SECURE_STORE: elephantsql-d7d89
services:
  - elephantsql-c6c60
  - elephantsql-d7d89
```
But this introduces a separate configuration and leads to duplication (of instance name).

## Proposal

Cloud Foundry can introduce a name for each service binding. This name should be prescribed by each application and should convey the purpose of the service binding.

For example using CF CLI:
```sh
cf bind-service my-app elephantsql-c6c60 regular-store
cf bind-service my-app elephantsql-d7d89 secure-store
```
or in manifest.yml
```
services:
  regular-store: elephantsql-c6c60
  secure-store: elephantsql-d7d89
```
To preserve compatibility, service binding name can be added as a new property in *VCAP_SERVICES*.
```
VCAP_SERVICES=
{
  "elephantsql": [
    {
      "name": "elephantsql-c6c60",
      "label": "elephantsql",
      "bind_name": "regular-store",
      ...
    },
    {
      "name": "elephantsql-d7d89",
      "label": "elephantsql",
      "bind_name": "secure-store",
      ...
    },
    ...
  ],
  ...
```

Ideally service binding names should be used as keys in *VCAP_SERVICES* so applications can easily find them.
But this would break compatibility with existing applications.
```
VCAP_SERVICES=
{
  "regular-store": {
    "name": "elephantsql-c6c60",
    "label": "elephantsql",
    ...
  },
  "secure-store": {
    "name": "elephantsql-d7d89",
    "label": "elephantsql",
    ...
  },
  ...
}
```

## Alternative Solutions

### Service Instance Tags

As Greg Cobb suggested in the cf-dev mailing list, one could use tags at service instance level
```sh
cf create-service elephantsql turtle elephantsql-c6c60 -t "db, regular-store"
cf create-service elephantsql turtle elephantsql-d7d89 -t "db, secure-store"
```
The tags can be updated veen after the service instance is created
```sh
cf update-service elephantsql-c6c60 -t "regular-store"
cf update-service elephantsql-d7d89 -t "secure-store"
```
Then an app could lookup a service by a specific tag. Tag could be hard-coded in the app. Since a service instance can be annotated with multiple tags, it can be bound to multiple apps.

#### Issues

There are several issues with this approach:
* [User-provided services][2] have no tags
* Ambiguous. Tags are not unique so it is possible that the same tag appears in multiple service instances bound to the same app
* It is possible that different apps put different meaning in the same tag. This can lead to problems if a service instance with this tag is bound to these apps.


[1]: http://docs.cloudfoundry.org/devguide/services/application-binding.html
[2]: https://docs.cloudfoundry.org/devguide/services/user-provided.html
[VCAP_SERVICES]: http://docs.cloudfoundry.org/devguide/deploy-apps/environment-variable.html#VCAP-SERVICES
