# Deployment Guide

To deploy, first clone the repository locally: `git clone git@gitlab.com:charts/gitlab.git`

Before running `helm install`, you need to make some decisions about how you will run GitLab.
Options can be specified using helm's `--set option.name=value` command line option.
A complete list  of command line options can be found [here](./command-line-options.md).
This guide will cover required values and common options.

## Selecting configuration options

In each section collect the options that will be combined to use with `helm install`.

### Secrets

There are some secrets that need to be created (e.g. ssh keys). By default they will be generated automatically, but if you want to specify them, you can follow the [secrets guide](secrets.md).

### Networking and DNS

By default, the chart relies on Kubernetes `Service` objects of `type: LoadBalancer`
to expose Gitlab services using name-based virtual servers configured with`Ingress`
objects. You'll need to specify a domain which will contain records to resolve
`gitlab`, `registry`, and `minio` to the appropriate IP for your chart.

*Include these options in your helm install command:*
```
--set global.hosts.domain=example.local
```

#### Dynamic IPs with external-dns

If you plan to use an automatic DNS registration service like [external-dns](https://github.com/kubernetes-incubator/external-dns),
you won't need any additional configuration.

If you provisioned a GKE cluster using the scripts in this repo, [external-dns](https://github.com/kubernetes-incubator/external-dns)
is already installed in your cluster.

#### Static IP

If you plan to manually configure your DNS records they should all point to a
static IP. For example if you choose `example.local` and you have a static IP
of `10.10.10.10`, then `gitlab.example.local`, `registry.example.local` and
`minio.example.local` should all resolve to `10.10.10.10`.

If you are using GKE, there is some documentation [here](../cloud/gke.md#creating-the-external-ip)
for configuring static IPs and DNS. Consult your Cloud and/or DNS provider's
documentation for more help on this process.

*Include these options in your helm install command:*
```
--set global.hosts.externalIP=10.10.10.10
```

### Persistence

By default the chart will create Volume Claims with the expectation that a dynamic provisioner will create the underlying Persistent Volumes. If you would like to customize the storageClass or manually create and assign volumes, please review the [storage documentation](storage.md).

### TLS certificates

You should be running GitLab using https which requires TLS certificates. By default the
chart will install and configure [cert-manager](https://github.com/jetstack/cert-manager)
to obtain free TLS certificates.
If you have your own wildcard certificate, you already have cert-manager installed, or you
have some other way of obtaining TLS certificates, [read about more TLS options here](./tls.md).

For the default configuration, you must specify an email address to register your TLS
certificates.
*Include these options in your helm install command:*
```
--set certmanager-issuer.email=me@example.local
```

### Postgresql

By default this chart provides an in-cluster postgresql database. This
configuration should not be used in production.

You can read more about setting up your production-ready database in the [advanced database docs](../advanced/external-db/README.md).

If you have an external postgres database ready,

*Include these options in your helm install command:*
```
--set global.psql.host=production.postgress.hostname.local
--set global.psql.password.secret=kubernetes_secret_name
--set global.psql.password.key=key_that_contains_postgres_password
```

### Redis

By default we use an single, non-replicated Redis instance. If desired, a highly available redis can be deployed instead. You can learn more about configuring: [Redis](../charts/redis) and [Redis-ha](../charts/redis-ha).

*To deploy `redis-ha` instead of the default `redis`, include these options in your helm install command:*
```
--set redis.enabled=false
--set redis-ha.enabled=true
```

### Outgoing email

By default outgoing email is disabled. To enable it, provide details for your SMTP server
using the `global.smtp` and `global.email` settings. You can find details for these settings in the
[command line options](command-line-options.md#email-configuration).

If your SMTP server requires authentication make sure to read the section on providing
your password in the [secrets documentation](secrets.md#smtp-password).
You can disable authentication settings with `--set global.smtp.authentication=""`.

If your Kubernetes cluster is on GKE, be aware that smtp [ports 25, 465, and 587
are blocked](https://cloud.google.com/compute/docs/tutorials/sending-mail/#using_standard_email_ports).

### Deploy the Community Edition

By default, the Helm charts use the Enterprise Edition of GitLab. If desired, you can instead use the Community Edition. Learn more about the [difference between the two](https://about.gitlab.com/installation/ce-or-ee/).

*To deploy Community Edition, include these options in your helm install command:*
```
--set gitlab.migrations.image.repository=registry.gitlab.com/gitlab-org/build/cng/gitlab-rails-ce
--set gitlab.sidekiq.image.repository=registry.gitlab.com/gitlab-org/build/cng/gitlab-sidekiq-ce
--set gitlab.unicorn.image.repository=registry.gitlab.com/gitlab-org/build/cng/gitlab-unicorn-ce
```

## Deploy using helm

Once you have all of your configuration options collected, we can get any dependencies and
run helm. In this example, we've named our helm release "gitlab".

```
helm dependencies update
helm upgrade --install gitlab . \
  --timeout 600 \
  --set global.hosts.domain=example.local \
  --set global.hosts.externalIP=10.10.10.10 \
  --set certmanager-issuer.email=me@example.local
```

## Monitoring the Deployment

This will output the list of resources installed once the deployment finishes which may take 5-10 minutes.

The status of the deployment can be checked by running `helm status gitlab` which can also be done while
the deployment is taking place if you run the command in another terminal.

## Initial login

You can access the GitLab instance by visiting the domain specified during
installation. If you manually created the secret for initial root password, you
can use that to sign in as `root` user. If not, Gitlab would've automatically
created a random password for `root` user. This can be extracted by the
following command (replace `<name>` by name of the release - which is `gitlab`
if you used the command above)

```
kubectl get secret <name>-gitlab-initial-root-password -ojsonpath={.data.password} | base64 -d
```

[secret-gl-certs]: secrets.md#gitlab-certificates
[secret-reg-certs]: secrets.md#registry-certificates
