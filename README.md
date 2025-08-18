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
* Register backup cluster - [register_backup_cluster.yaml](./register_backup_cluster.yaml)
* Autopilot cluster resize - [autopilot_volume_resize.yaml](./autopilot_volume_resize.yaml)
* Node failure resilience test - [resilience_node_failure.yaml](./resilience_node_failure.yaml)

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

### Backup Cluster Add

Using the inputs at [backup_cluster_register.yaml](./sample_inputs/backup_cluster_register.yaml), you can create new service accounts for backup to use and register the cluster with the backup UI.

```console
$ ansible-playbook backup_cluster_register.yaml -e @<vars_file>
```

### Autopilot volume resize

This module requires no inputs, but follows the basic POC docs found at https://docs.portworx.com/poc/autopilot_volume-resize to
demonstrate a volume resize in the current cluster context.

```console
$ ansible-playbook autopilot_volume_resize.yaml
```