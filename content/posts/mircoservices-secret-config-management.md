+++
title ="Microservices Secrets/Configuration Management"
date = "2018-07-10"

[taxonomies]
tags=["security", "aws"]
+++

> Your configuration management strategy will determine how you manage all of
> the changes that happen within your project. It thus records the evolution of
> your systems and applications. It will also govern how your team
> collaborates—a vital but sometimes overlooked consequence of any configuration
> management strategy.
>
> -- Continuous Delivery: Reliable Software Releases through Build, Test, and
> Deployment Automation

Configuration management is one of the critical component of any deployment.
Effective config management can greatly simplify your deployments. Ideally code
should be configurable enough that it doesn't hardcode any configuration
(Although this is hard to get it right the first time, eventually you'll figure
out what should be configurable). Configuration, on the other hand, can be plug
and played for different type of deployment. For example, in case of rails app,
staging apps will need to read/write from staging database, and production apps
from production. [12 factor app](https://12factor.net/config) recommends that
app reads configuration from environment variables, as they are both language
and platform agnostic.

Here are some ideal features any configuration management system should strive
for:

- centralized, you use same mechanism to read secrets in pipeline jobs, in
  staging env or in production env, pretty much everywhere you need them
(However, it can be good idea to separate production & staging configuration).
- rollbackable, you should be able to revert a config change back, ideally you
  shouldn't lose any info, doesn't matter what you change.
- encrypted, you should never store secrets in plain text. You should rotate the
  key frequently.
- auditing, you should be able to tell who changed what and when, when things go
  south
- principle of least privilege, service A shouldn't be able to read secrets of
  service B, likewise developer of service A should only have read write access
to secrets of service A.

# AWS Parameter Store

AWS Parameter Store is a simple service which stores strings either as plain
text or encrypted. It has all good features of a centralized config management
system. For example, 
- you can encrypt the values using custom KMS keys. 
- you can track the changes made overtime. 
- you can have fine grain control of who has what access (read or write) based
  on IAM policies. 

And since it's AWS managed, you don't have to worry about hassle of managing yet
another service. 

# Chamber

That said, it's not that user friendly to use, either via console or cli
(Atleast was by the time of writing this post). Segmentio has built a cli tool
around it, known as [chamber](https://github.com/segmentio/chamber). The
interface is really simple & powerful to use. For example, it can export all the
secrets as environment variables before booting up application, which is nice.

Chamber by default requires a custom KMS key (for encryption of values). It
needs to be named spefically `parameter_store_key`. You can create it by running
following terraform script (or equivalent steps in console or cli):

```
resource "aws_kms_key" "parameter_store" {
  description             = "Parameter store kms master key"
  deletion_window_in_days = 10
  enable_key_rotation     = true
}

resource "aws_kms_alias" "parameter_store_alias" {
  name          = "alias/parameter_store_key"
  target_key_id = "${aws_kms_key.parameter_store.id}"
}
```

Chamber segregates configuration based on services. For example, you can create
following services:

```
# for brevity, this shows just one key, you can write multiple key values for
each service by repeating the writes

# microservice a, contains only application config
chamber write my_app_a app myapp-a

# microservice b
chamber write my_app_b app myapp-b

# ci database, contains database connection config
chamber write mysql_db_ci host ci_host

# stag database
chamber write mysql_db_staging host stag_host
```

Each of your centralized datastores (RDS, ES, redis) can be separate services.
Now, you can, run following:

```
# to boot app with ci-db in CI pipelines
chamber exec my_app_a mysql_db_ci -- bundle exec rails test

# to boot app in staging
chamber exec my_app_b mysql_db_staging -- bundle exec rails server
```

You can plug `mysql_db_ci` or `mysql_db_staging` as needed. If you use docker,
you can add this in Dockerfile:

```
CMD ["chamber", "exec", "my_app_a", "mysql_db_ci", "--", "bundle exec rails s"]
```

When you specify multiple services, chamber fetches keys of all services in
order, merges them (later service key's value overriding former in case of
conflict). Finally it's boots the application as it's sub process with all env's
set. It also acts like a dumb init, and passes all signals to application
process. 

IAM policies can be used to ensure principle of least priviledges. There are
many ways to attach IAM role to your application container. If you're running
kubernetes, checkout [kube2iam](https://github.com/jtblin/kube2iam) project. Or
you can use [EC2 Container Service
Agent](https://github.com/aws/amazon-ecs-agent) (ecs-agent).

Following policy ensures that container is only able to read secrets for service
`{{ service_name }}`

```
{
    "Sid": "",
    "Effect": "Allow",
    "Action": [
        "ssm:GetParameters",
        "ssm:GetParametersByPath"
    ],
    "Resource": "arn:aws:ssm:*:*:parameter/<< service_name >>/*"
},
```

# Mozilla Sops

This system is good by itself. However, overtime, we noticed that managing keys
(adding or updating) via chamber was not that user friendly.

Chamber supports export & import of keys from json file. So instead of
reading/writing single keys, we created a json file per service and stored them
in a common repository:

```
├── production
│   ├── .sops.yaml
│   ├── grafana
│   │   ├── grafana_app.json
│   └── metabase
│       └── metabase_app.json
└── staging
    ├── .sops.yaml
    ├── sentry
    │   └── sentry_app.json
    └── gitlab
        └── gitlab_app.json
```

The data is encrypted using [Mozilla Sops](https://github.com/mozilla/sops)
(which is really cool, you can use your favourite editor, also it encrypts only
the values, so that git diffs are meaningful). Sops supports many types of
encryption, we use AWS KMS, because it's simpler to manage. 

If you add following config in root dir, sops can use that KMS key by default
for all new files (in this case we use single KMS key for all staging services):

```
$ cat staging/.sops.yaml
creation_rules:
  - path_regex: \.json$
    kms: '<< arn_of_kms_key >>'
```

You'll have to create KMS key in AWS (see above on how to do that, also checkout
readme of sops).

Now you can create new service as

```
sops staging/gitlab/gitlab_app.json
```

This will open editor, you can update the key & values here. If you open this
file via normal editor, you can see that all values are encrypted.

```
    "access_token":
"ENC[AES256_GCM,data:C+daldrNWpQhDUolPjhM9yyAERuxylfycA56wOoe5Vs=,iv:5Lss6SqmE23fQ3YA7T97V+v0mXd/cDNgM3FS24x/JdE=,tag:tbP2quqao4I4D54HZhyFlw==,type:str]",
```

Finally following script syncs these keys to parameter store using chamber on
each commit via pipeline job:

```
#! /bin/bash

set -o errexit

ACCOUNT=$1

if [[ ! "$ACCOUNT" =~ (production|staging) ]]; then
  echo "error: param-sync: invalid account. Usage: $0 production|staging"
  exit 1
fi

synced() {
  git tag "$ACCOUNT-synced" -f
  git push origin "$ACCOUNT-synced" -f
}

services_changed() {
  git diff --name-status "$ACCOUNT-synced..master" | cut -f2 | grep "$ACCOUNT" |
grep '.json$'
}

NO_SERVICES_CHANGED=$(services_changed | wc -l)

echo "$NO_SERVICES_CHANGED services changed:"
services_changed || true

if [[ "$NO_SERVICES_CHANGED" == "0" ]]; then
  echo "Skipping sync..."
else
  for SERVICE_FILE in $(services_changed); do
    SERVICE="$(basename "$SERVICE_FILE" | cut -f 1 -d '.')"
    echo "$SERVICE:"
    sops -d "$SERVICE_FILE" | chamber import "$SERVICE" -
  done
fi

echo "All changes synced. Tag current commit in git"
synced
```

# References

- [The right way to manage secrets with
  AWS](https://segment.com/blog/the-right-way-to-manage-secrets/)
