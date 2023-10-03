# exited-container-pruning
Prune exited container per openshift worker nodes
The cronjob keep 2000 exited container per node
If more than 2000 container existe in a node, the cronjob will prune the extra starting by the oldest exited container.
The 2000 is just a number and  could be changed easily in the cronjob

## order of manifest execution
1. oc create -f manifest.yaml

2. oc adm policy add-scc-to-user privileged -z openshift-cleanup

3. oc create -f cronjob.yaml
