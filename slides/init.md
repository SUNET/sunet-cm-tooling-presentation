---
title: Multiverse, cosmos and puppet @Sunet
author: jocar
theme:
  # Theme can be get here:
  # https://github.com/SUNET/presenterm-sunet-theme
  name: sunet
---
Overview
---
When we at Sunet refer to _cosmos_ we mean three components
<!-- pause -->
* Multiverse, the repo which contains all configuration and some tooling
* * Often referred as _ops repo_, e.g "eduid-ops"
* * Should be merged with upstream on a "regular basis"
<!-- pause -->
* Cosmos, a wrapper to sync and verify the ops repo to the host. It's also responsible for execution of the configuration management software
<!-- pause -->
* Puppet, the current configuration management software used within the multiverse
<!-- pause -->

# Links
* [multiverse](https://github.com/SUNET/multiverse)
* [cosmos](https://github.com/SUNET/cosmos-dpkg)
* [puppet](https://www.puppet.com/docs/puppet/7/puppet_index.html)

<!-- end_slide -->

Multiverse - tools
---
There are a few commands and scripts in the root of the ops repo which we use to handle host(s) and the repo.
<!-- pause -->
# ./prepare-iaas-(debian|ubuntu)

script(s) prepare a host delivred from our SaaS provider for _cosmos_. Adds a few depends, configure network and disables unnessesary users.
<!-- pause -->
# ./addhost -b \<fqdn\>
script to bootstrap a machine and configure it to use cosmos.
<!-- pause -->
# ./edit-secrets \<fqdn\>
Handle the secrets for a given host, more on that later.
<!-- pause -->
# ./host-puppet-conf-test \<fqdn\>
script used to test _local_ uncommited puppet classes on a given host. not so widly used any more since most tries out their changes directly on a dev machine.
<!-- pause -->
# ./bumptag
Verifies *your* trust against the latest tag (and some commits on important files) and creates a signed tag. Also responsible of pushing the tag to the git server.
<!-- end_slide -->
Multiverse - overlays
--- 
Some paths in the ops repo are handled in a special way
<!-- pause -->
# global/
Applies to all hosts in the ops repo
<!-- pause -->
## global/overlay/
A tree structure that will be rsynced with / as base. E.g:
* global/overlay/usr/local/bin/sunet-fleetlock
* global/overlay/etc/cosmos/keys/jocar-13376BF892B5871181A218E9BE4EC2EEADF2C31B.pub

will end up in

* /usr/local/bin/sunet-fleetlock
* /etc/cosmos/keys/jocar-13376BF892B5871181A218E9BE4EC2EEADF2C31B.pub
<!-- pause -->
## global/delete/
A tree structure with the opposite effect as **_overlay_**. All files (or skeletons) in the structure will be removed when cosmos is run

<!-- pause -->
# <some_service>-common/
It is possible to configure a host to use a shared overlay between some selected hosts. E.g the mysql cluster host might have some shared files between them. The files would be put in this overlay. Same stucture as **_global/_** with a **_overlay_** and **_delete_**.

<!-- pause -->
# \<fqdn\>/
The default overlay for a given host (created by `./addhost`). In here we store host specific files and secrets. Same structure as **_global/_** with a **_overlay_** and **_delete_**.
<!-- end_slide -->
Multiverse - configuration
--- 
# cosmos.conf (in repo)
Used by `./addhost` to configure the default repo url (git) and which tag pattern cosmos should use.
<!-- pause -->
# global/overlay/etc/puppet/setup_cosmos_modules (most often available)
Python script executed by cosmos on the host to determine which puppet modules and corresponding tag the host should use. Very useful to configure a feature branch on a set of hosts or just a single host. E.g
```python
if "monitor-2.test" in host_info["fqdn"]:
  modules["sunet"]["tag"] = "testing-2*"

if "bankid" in host_info["fqdn"]:
  modules["sunet"]["tag"] = "jocar-feature-xyx-2*"
```
<!-- pause -->
# global/overlay/etc/puppet/cosmos-rules.yaml (symlinked to cosmos-rules.yaml)
Yaml file to configure which puppet class(es) to apply the hosts within a given ops repo. Parsed by `re.match()` with the host's fqdn to match against.

E.g
```yaml
# Applies on all hosts
'.+':
   sunet::server:
      sshd_config: true

# Applies on one host
'web-server-1.example.com':
   sunet::dockerhost2:
```
<!-- end_slide -->

Multiverse - trust
---
All commits within a ops repo *SHOULD* be signed with a GPG Key¹

All tags within a ops repo *MUST* be signed (handled by `./bumptag`) with a GPG key¹

Each ops repo has it's own trust store. The keys are stored in **_global/overlay/etc/cosmos/keys/_** and _SHOULD_ be named "\<name|uid\>-\<the long fingerprint of key\>.pub", e.g "jocar-13376BF892B5871181A218E9BE4EC2EEADF2C31B.pub"

Besides trusting the current ops repo the trusted keys are used to validate incoming external puppet modules.

¹ The GPG key *MUST* be stored on a Yubikey created according to [token-setup.md](https://github.com/SUNET/eduid-docs/blob/master/token-setup.mkd)

<!-- end_slide -->

Cosmos
---

Cosmos is the wrapper that is executed from each host to fetch and apply the configuration in multiverse. If cosmos is not disabled (more on that later) it will be run by cron every 15 minutes. It's only the host itself that can trigger a execution as there are no APIs or listeners to a master of any kind.

<!-- end_slide -->

Cosmos - configuration
---
There is not much to configure regarding cosmos but…

<!-- pause -->

# /etc/cosmos.conf (on host)
Created by `./addhost` and templated with the configuration from the **_cosmos.conf_** in the ops repo. Here we could configure which tag that should be used for the ops repo (COSMOS_UPDATE_VERIFY_GIT_TAG_PATTERN). Shared overlays might also be automatically added if they match a fixed set of rules based on the hostname. Since Sunets naming schema changed over time the automatic discovery of shared overlays most often doesn't work which makes it manual labour.

Example configuration:
```sh
COSMOS_REPO_MODELS="$COSMOS_REPO/web-server-1.example.com/:$COSMOS_REPO/mysqlcluster-test-common/:$COSMOS_REPO/global/"
COSMOS_UPDATE_VERIFY_GIT_TAG_PATTERN="cosmos-ops*"
```

<!-- pause -->
# /etc/no-automatic-cosmos
If this file is present no execution of cosmos from cron will happen. It's still possible to run cosmos by hand. It's good practice to add a name an reason to why cosmos is disabled in the file.

<!-- end_slide -->

Cosmos - execution
---
Ways to make things happen

<!-- pause -->
# run-cosmos
In order to execute cosmos manually from the host you can run `run-cosmos` ideally with the flag `-v` the get more output of the command. Be ware of warnings from puppet. 

<!-- pause -->
# puppet apply
`run-cosmos` fetches the ops repo and external puppet modules on each run which can be a bit irritating during development of manifests or other configuration. By disabling cosmos (/etc/no-automatic-cosmos) changes can be made to local files and manifests with anyone interfering. In this mode you can run `puppet apply -v /etc/puppet/manifests/cosmos-site.pp` in order to execute the puppet configuration on disk. Add `-d` for a very verbose output.

The puppet modules can be edited in **_/etc/puppet/modules/_** or in **_/etc/puppet/cosmos-modules/_**. Any uncommited changes will be overwritten on next `run-cosmos`.
<!-- end_slide -->
puppet -- modules
---
A puppet module is a set of classes. We fetch modules from three sources

<!-- pause -->
# Sources
## apt
Some standard modules are installed and handled via `apt` and maintaned by the distro.

<!-- pause -->
## local modules in ops repo
It is possible to create a number of local repos in **_global/overlay/etc/puppet/modules/\<your module name\>_**. Local modules and classes are not as common as before but they still exists for local and niche needs.
<!-- pause -->
## External modules via git
Via `setup_cosmos_modules` any number of external repos can be added. Nowadays we mostly only use "nagioscfg" for creating monitoring server configuration and "sunet" where we put everything else that can benifit most ops repos at Sunet. Cosmos handles the freshness of the repos and verifiy that the latest tag is signed by someone how it trusts.

# Layout
A minimal layout for a puppe module would look like
```
<module-name>/manifests/init.pp
<module-name>/manifests/<class_name>.pp
<module-name>/manifests/<another_class_name>/<class1>.pp
<module-name>/manifests/<another_class_name>/<class2>.pp
<module-name>/templates/<class_name>/template_used_by_class.yaml.erb
```
Puppet will invoke `init.pp` if e.g "module-name" is called for or "class_name.pp" if "module-name::class_name" was called for.
<!-- end_slide -->

puppet - variables and secrets
---

puppet can fetch variables from multiple sources.

<!-- pause -->
# facts
Built-in facts regarding the OS, hardware and much more. Can be anything from SSH keys to hard drives. It's also possible to add you own facts.

<!-- pause -->
# Arguments to the class
If a class accepts arguments they can be specified in **cosmos-rules.yaml**

<!-- pause -->
# Hiera
Hiera is a variable and secrets store for puppet. It's yaml based configuration files. E.g

```yaml
nrpe_clients:
   - 2001:6b0:40::21c
   - 89.45.236.45
```

<!-- pause -->
## /etc/hiera/data/common.yaml
Stored in global overlay and contains variables useful for the whole ops repo. Eg. ip adresses of monitor servers or which ssh keys should be accepted
<!-- pause -->

## /etc/hiera/data/group.yaml
Stored in shared overlay and contains variables useful for the selected group of servers
<!-- pause -->

## /etc/hiera/data/local.yaml
Host specific variables
<!-- pause -->

## /etc/hiera/data/local.eyaml
Host specific secrets. The file is handled from the ops repo with the `./edit-secrets` command, e.g `./edit-secrets web-server-1.example.com`. The command will ssh in to the selected machine and spawn an unencrypted editor. When the editor is closed with a changed file an encrypred eyaml file will be returned and added to the ops repo. The keys to decrypt the file is only available on the host itself. The syntax looks a little bit different since the secrets are wrapped in _DEC::PKCS9[my-secret]_ but otherwise it's an ordinary yaml
<!-- end_slide -->


TLS
---
TLS at Sunet can be done in multiple ways. Some may services are placed behind a load balancer (which handles the public TLS termintion) while others exposes port 443 (or other) directly to the world.

<!-- pause -->

## Infra cert
Our internal CA. Mostly used between server and loadbalancer to secure the trafic but also used for machine to machine authenication. E.g Redis/Redict.

CSRs are created in nunoc-ops with `scripts/mkreq` and signed certs can be found here: [http://ca.sunet.se/infra/](http://ca.sunet.se/infra/). [sunet::ici_ca::rp](https://github.com/SUNET/puppet-sunet/blob/main/manifests/ici_ca.pp) is a great companion when working with Infra certs.

Documentation in the [wiki](https://wiki.sunet.se/display/sunetops/SUNET+CA).


<!-- pause -->

## ACME C
Mostly used with the puppet class [sunet::dehydrated::client](https://github.com/SUNET/puppet-sunet/blob/main/manifests/dehydrated/client.pp). HTTP Challanges are initiated and proxied to from the central acme c host (acme-c.sunet.se). Each service using AMCE C must be publicly available in order for the verification to work. Besides adding a new domain to DNS, configuration is needed in the acme-c server (via nunoc-ops) to getting started with a new domain.

Read more about it in the [wiki](https://wiki.sunet.se/display/sunetops/Acme-c+technical+documentation)

<!-- pause -->

## ACME D
ACME Challanges thought DNS so the service don't need to be publicly available. The puppet class [sunet::certbot::acmed](https://wiki.sunet.se/display/sunetops/sunet%3A%3Acertbot%3A%3Aacmed) is recommended in order the get certificates with ACME D. AMCE D requires a special DNS entry and configuration in the local ops repo in order to get started with a new domain.

Read more about it in the [wiki](https://wiki.sunet.se/display/sunetops/acme-d.sunet.se)

<!-- end_slide -->

bonuses
---

# scriptherder
We wrap all jobs run in cron in a wrapper called `scriptherder`. Scriptherder captures the output and exit values of the scripts which is monitored by our monitors. To view the current status of all scriptherder configured jobs you can run `scriptherder ls`.
<!-- end_slide -->
