# Puppet Master stack

This stack installs and configures a full [Puppet Master](https://puppetlabs.com/puppet/puppet-open-source) stack.

This includes:

* Puppet Server
* Puppet CA
* R10K
* PuppetDB
* ActiveMQ
* Puppetboard

## Puppetserver service

This service is set up to scale, and does not behave as a CA. It takes several parameters:

* the `dns_alt_names`, which must be provided if you want to use a CNAME for the service
* the private GPG key if you want to use hiera-eyaml
* the `hiera.yaml` content


## R10K service

The R10K service is comprised of two services in the stack:

* an `r10k` service, which is a sidekick on each `puppet` server container
* a `webhook` service, which exposes the R10K service to the world

The setup is based on [Mcollective](https://forge.puppetlabs.com/zack/r10k).

The `webhook` service provides a service listening on port 9000. It is currently set up to receive a GitHub webhook payload on `/hooks/r10k`.

If you don't want to host your r10k repository on GitHub, you can use the `puppet-git-repository` template, which sets up a git repository with a post-receive hook to a GitHub-compatible webhook.

When the r10k webhook is triggered, it pings all `r10k` services so they pull from the git repository to populate the code repositories of each Puppet server.


## Certificate authority

In order to allow the Puppetserver containers to scale, the CA has been split from the compile component.

### CA port

The Puppetserver (compile component) and the CA both use a Puppetserver image and listen on port 8140 by default.

In order to avoid port interlock, the default port for the CA in this stack is 8141 instead.

This means your Puppet nodes need to override two settings:

```inifile
[main]
ca_server = puppetca.puppet.<env>.<domain>
ca_port = 8141
```

See the Dynamic DNS section for more information about server names.

You can use a CNAME pointing to the CA service. In this case, you must pass this name in the `dns_alt_names` field for the CA (or provide your own key and certificate generated for this name).


### Certificate names

Container names in the stack are generated by Rancher. The stack does not fix the host names nor the certificate names. Instead, it uses the container hostname, which is garanteed to be unique, regardless of how many instances of the service are running.

### Autosign policy

In order to fully automate the stack deployment, the CA is set up to autosign certificates generated by the other services in the Rancher stack.

This feature relies on using the [rancher-metadata](http://docs.rancher.com/rancher/metadata-service/) to pass secrets in the CSR. The passed secret consists of a combination of the service name and the Rancher UUID for the service.

### CA persistence

In order to keep the setup simple, this stack does not retain CA data. This means the CA is regenerated every time the CA container is restarted.

This means it is currently not possible to revoke certificates in this setup.

You can pass a CA key and CA certificate to the stack to ensure they do not change. If you do not pass them, a new one will be regenerated *at every container creation* (including rolling upgrades).


## Dynamic DNS

All services are made to scale out on Rancher nodes, and can thus end up on various IP addresses.

The recommended way to access these services is to setup [a Dynamic DNS service](http://docs.rancher.com/rancher/rancher-services/dns-service/).
