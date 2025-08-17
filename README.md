# Portworx Ansible Playbooks

Deploying Portworx is often significantly dependent on the installation context.
In order to deal with this complexity, I am trying to simplify the standard install
and update operations into a simple playbook.

The goal of this repository is to enhance the Portworx POC process to ensure
ease of use and long term automation capabilities.

## Current Workflows

The following workflows are supported:

* Async DR setup two clusters - [pxdr_setup.yaml](./pxdr_setup.yaml)
    * Tear down DR setup - [remove_clusterpair.yaml](./remove_clusterpair.yaml)


### Async DR Setup/Teardown

The variable inputs used for this process can be found at [dr_installation.yaml](./sample_inputs/dr_installation.yaml).

To pair two clusters, run:

```console
$ ansible-playbook pxdr_setup.yaml -e @<vars_file>
```

To tear down the created clusterpair, run:

```console
$ ansible-playbook remove_clusterpair.yaml -e @<vars_file>
```