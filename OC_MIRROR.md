Step 1:

Download the necessary binaries from:
https://mirror.openshift.com/pub/openshift-v4/clients/ocp/4.20.20/


Can replace ‘/4.20.20/ with the desired version of OpenShift.

Download binaries modified as needed for your hardware setup (ie arm, amd64, rhel9, etc):

oc-mirror.rhel9.tar.gz

openshift-client-linux-amd64-rhel9-4.20.20.tar.gz


Download the latest oc-mirror registry:

https://mirror.openshift.com/pub/cgw/mirror-registry/latest/mirror-registry-amd64.tar.gz



Step 2:

Extract the binary files:

tar -xvf openshift-client-linux-amd64-rhel9-4.20.20.tar.gz

tar -xvf mirror-registry-amd64.tar.gz


Step 3:

Move the binary files to /usr/local/bin

sudo mv oc kubectl oc-mirror /usr/local/bin


Step 4:

Chmod oc-mirror

sudo chmod +x /usr/local/bin/oc-mirror


Step 5 (optional):

Verify versions:

oc version (should match client version you downloaded above)

oc-mirror version


Step 6:

Copy pull secret to host:

mkdir ~/.docker

cat pullSecret >> ~/.docker/config.json


Step 7:

Create the imageset-config.yaml file:

vim imageset-config.yaml

Note: Run this command to see a list of available operators:
oc-mirror list operators --catalog registry.redhat.io/redhat/redhat-operator-index:v4.20


(sample file:)


1 kind: ImageSetConfiguration

  2 apiVersion: mirror.openshift.io/v2alpha1

  3 mirror:

  4   platform:

  5     channels:

  6       - name: stable-4.20

  7         minVersion: 4.20.20

  8         maxVersion: 4.20.23

  9     graph: true

 10   additionalImages:

 11     - name: registry.redhat.io/ubi8/ubi:latest

 12     - name: registry.redhat.io/ubi9/ubi:latest

 13   operators:

 14     - catalog: registry.redhat.io/redhat/redhat-operator-index:v4.20

 15       packages:

 16         - name: kubernetes-nmstate-operator

 17         - name: cluster-logging

 18         - name: compliance-operator

 19         - name: local-storage-operator

 20         - name: ansible-automation-platform-operator

 21         - name: openshift-cert-manager-operator

 22         - name: rhacs-operator

 23         - name: kubevirt-hyperconverged

 24         - name: openshift-gitops-operator

 25         - name: cincinnati-operator

 26         - name: file-integrity-operator

 27         - name: lvms-operator

 28         - name: odr-hub-operator

 29         - name: cluster-observability-operator

 30         - name: node-healthcheck-operator

 31         - name: node-maintenance-operator

 32         - name: node-observability-operator

 33         - name: cluster-kube-descheduler-operator

 34         - name: mtv-operator

 35         - name: kiali-ossm

 36         - name: machine-deletion-remediation

 37         - name: openshift-custom-metrics-autoscaler-operator

 38         - name: openshift-external-secrets-operator

 39         - name: openshift-zero-trust-workload-identity-manager

 40         - name: self-node-remediation

 41         - name: servicemeshoperator3

 42         - name: vertical-pod-autoscaler

 43         - name: web-terminal

 44         - name: deployment-validation-operator

 45         - name: rhbk-operator

 46         - name: loki-operator

 47     - catalog: registry.redhat.io/redhat/certified-operator-index:v4.20

 48       packages:

 49         - name: portworx-certified


Step 8:

Create a directory to ignore signatures from certified-catalog:

sudo mkdir -p /etc/containers/registries.d/


Step 9:

Create a file named /etc/containers/registries.d/registry.connect.redhat.com.yaml and paste this inside:


docker:

  registry.connect.redhat.com:

    use-sigstore-attachments: false



Step 10:

Pull down the bundle:

oc-mirror --v2 -c imageset-config-4.20.20.yaml --cache-dir=/path-to-large-drive/bundle-cache file://./mirror-4.20.20 --retry-times 60 --image-timeout 60m --parallel-images 4 --retry-delay 5s
