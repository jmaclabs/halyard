# Halyard Deploys

## Purpose

Explain how Halyard handles deploying Spinnaker.

## Terminology

* **Deployment Type**: How Spinnaker is divided among all the instances it is
 running on. An example is having each Spinnaker service run a separate
 instance. See [Deployment Types](#deployment-types) section for more
 examples.
* **Service Dependencies**: Dependencies of Spinnaker that need to be run as
 stand-alone services. An example is Redis, a non-example is Packer.
* **Deployment Environment**: The infrastructure Spinnaker is running on,
 Spinnaker's Deployment Type, and Spinnaker's service dependencies.

## Deployment Types

* **LocalDebian**: This is every Spinnaker service installed as a Debian
 package on the node Halyard is running on.
* **LocalGit**: This is every Spinnaker service installed from Git on the 
 node Halyard is running on. This is intended for use by Developers.
* **Distributed**: This is every Spinnaker service installed remotely on a
 specified Cloud Provider as separate, independently scalable instance. There
 is no customization or breakout of specialized profiles for this deployment.

Every deployment type must address a few issues:

1. **Configuration**: How Spinnaker recieves the config it needs to run.
2. **Service Dependency Setup**: What services Halyard must install to get
 this deployment to run.
3. **Deployment & Updates**: How the Spinnaker services are deployed and
 updated in this environment.
4. **Logging & Monitoring**: How the monitoring daemon is instantiated &
 configured, and how third-party metric stores are configured.

### LocalDebian Deployments

> Timeline: Already included in Halyard.

Halyard handles this deployment type by generating an install script that
configures apt-sources, pins versions for each subcomponent, installs and runs
dependencies, and places generated config in the correct directories.

1. **Configuration**: The script generated by Halyard has commands to copy all
 configuration from the necesssary Spinnaker config already generated by 
 Halyard into `/opt/spinnaker/config`, `/opt/spinnaker-monitoring` and 
 `/etc/apache2`. Any required files are already in the correct location.
2. **Service Dependency Setup**: Service dependencies are to be handled on a
 case-by-case basis, having each generate its own install/update script inside
 of the `InstallableServiceProvider`.
3. **Deployment & Updates**: All spinnaker packages are pinned by writing
 `apt` preferences in `/etc/apt/preferences.d/pin-spin-{artifact}` for each
 Spinnaker artifact. During the initial install and subsequent updates, this
 file is always updated before attempting to grab new packages via `apt`.
4. **Logging & Monitoring**: When enabled, the daemon is downloaded &
 configured with a registry pointing at all local services. Dashboards are
 published using Eric's `spinnaker-monitoring-third-party` scripts.

### LocalGit Deployments

> Timeline: Late April

Halyard handles this deployment by installing all Spinnaker service git repos
in a user-configurable directory, with a user-configurable `upstream` and
`origin`.  There will be a special version called `HEAD` that supersedes any 
configuration in our profile registry that will take all services at git `HEAD` 
to use for config generation.

### Distributed Deployments

> Timeline: Varies by cloud provider. See below.

1. **Configuration**: Varies by cloud provider.
2. **Service Dependency Setup**: Varies by cloud provider.
3. **Deployent & Updates**: This deployment involves first deploying a separate 
 Orca, Clouddriver and Redis using their published `-bootstrap.yml` profiles to 
 only communicate with one-another, and then orchestrate the remainder of the 
 update/deployment through them by publishing red/black update pipelines that 
 redeploy desired services with a cluster history size of 2 for fast rollbacks.
 During this deployment Orca is given an unlimited cluster history to avoid
 pruning Orca instances still handling pipeline executions. Furthermore, 
 cache schema changes in Clouddriver are handled by flushing Redis after 
 scaling the old instance of Clouddriver down to 0 instances, to avoid
 competing cache writes with different schemas.
4. **Logging & Monitoring**: Varies by cloud provider.

#### Kubernetes 

> Timeline: Early April. The core features are in, but a few touchups as
> detailed in https://github.com/spinnaker/halyard/projects/1

1. **Configuration**: Halyard publishes all configuration as Kubernetes
 secrets, and describes to the individual services where to mount them.
2. **Service Dependency Setup**: The only external dependency is redis, and if
 it is not provided, Spinnaker will deploy a Redis instance.
3. **Deployent & Updates**: See above.
4. **Logging & Monitoring**: Monitoring daemon containers are attached as
 sidecars, and monitor the service they are attached to exclusively.
 Dashboards are published using Eric's `spinnaker-monitoring-third-party` 
 scripts.

#### GCE

> Timeline: Middle April.

1. **Configuration**: Halyard uses the configured
 [Vault](https://www.vaultproject.io/) storage and access 
 token to write all configuration as secrets per-instance, which the bootscript
 of the instance will download and place in their intended directories. The
 secret format looks roughly like this:
 ```
 /secret/spinnaker/clouddriver-{id}: {"configs": ["config1, "config2"]}
 /secret/spinnaker/clouddriver-{id}/config1: {"path": "/a/b", "contents" "xyz"}
 /secret/spinnaker/clouddriver-{id}/config2: {"path": "/z/y", "contents" "cba"}
 ```
 The instance will be provided with top-level secret id `clouddriver-{id}` and
 use that to download the secrets `config1` & `config1`, and place their
 `contents` into `path`.  Service endpoints will be configured using typical Consul DNS names, which
 will be written during config generation. Both Vault & Consul endpoints will
 be distributed as instance metadata.
2. **Service Dependency Setup**: Redis, Consul, & Vault will all be deployed if
 not supplied. A first-pass will configure a small Consul cluster and
 file-backed Vault data store for simplicity.
3. **Deployent & Updates**: See above.
4. **Logging & Monitoring**: Monitoring daemon debians are baked into the
 service's images, and are enabled via configuration. Dashboards are published 
 using Eric's `spinnaker-monitoring-third-party` scripts.
