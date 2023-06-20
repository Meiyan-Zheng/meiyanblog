---
layout: post
title: How to update configurations for existing overcloud with director operator?
---
1. Create new file `workaround.yaml` to pass new passwords: 
```
[root@dell-r640-009 deploy]# tree
.
|-- tripleo_deploy_tarball
|   |-- net-config-two-nic-vlan-computedpdk-qe.yaml
|   |-- net-config-two-nic-vlan-computedpdksriov-qe.yaml
|   |-- net-config-two-nic-vlan-computehci_leaf1.yaml
|   |-- net-config-two-nic-vlan-computehci_leaf2.yaml
|   |-- net-config-two-nic-vlan-computehci.yaml
|   |-- net-config-two-nic-vlan-compute_leaf1.yaml
|   |-- net-config-two-nic-vlan-compute_leaf2.yaml
|   |-- net-config-two-nic-vlan-computesriov-qe.yaml
|   |-- net-config-two-nic-vlan-compute.yaml
|   `-- roles_data.yaml
`-- tripleo_heat_envs
    |-- cloud-names.yaml
    |-- containers-prepare-parameter.yaml
    |-- debug.yaml
    |-- network-common.yaml
    |-- network-environment.yaml
    |-- register-nic-templates.yaml
    |-- selinux.yaml
    |-- storage-backend.yaml
    |-- tls-certs.yaml
    |-- tls.yaml
    |-- updates.yaml
    `-- workarounds.yaml <-------------------------------
2 directories, 22 files
```

2. The content of `workaround.yaml`:
```
[root@dell-r640-009 deploy]# cat tripleo_heat_envs/workarounds.yaml 
parameter_defaults:
  parameter: value
```

3. Delete existing configmap contains heat environment files and create a new one:
```
[root@dell-r640-009 config_generator_default]# cat openstackconfiggenerator.yaml 
apiVersion: osp-director.openstack.org/v1beta1
kind: OpenStackConfigGenerator
...
  heatEnvConfigMap: heat-env-config-deploy    // configmap contains heat environment files
  tarballConfigMap: tripleo-tarball-config-deploy
```
Delete configmap `heat-env-config-deploy` and create a new one with same name:
```
[root@dell-r640-009 config_generator_default]# oc delete cm heat-env-config-deploy
[root@dell-r640-009 deploy]# oc create cm heat-env-config-deploy --from-file=./tripleo_heat_envs
configmap/heat-env-config-deploy created
```

4. Delete existing openstackconfiggenerator and Create a new one:
```
[root@dell-r640-009 config_generator_default]# oc delete OpenStackConfigGenerator/default
openstackconfiggenerator.osp-director.openstack.org "default" deleted
[root@dell-r640-009 config_generator_default]# oc create -f openstackconfiggenerator.yaml 
openstackconfiggenerator.osp-director.openstack.org/default created
[root@dell-r640-009 config_generator_default]# oc get OpenStackConfigGenerator
NAME      STATUS
default   Initializing
```

5. Confirm new configVersion and update openstackdeploy:
```
[root@dell-r640-009 deploy_default]# oc get -n openstack --sort-by {.metadata.creationTimestamp} osconfigversions -o json
...
host\n@@ -818 +864 @@   strategy: tri",
                "hash": "n67dh579hf8h65h545h66hb7h689h5c9h89hf4h648h79h5ch57h567hbfh77h5d6h658h559hcch579h5bh9fhb5h65fh55ch5bfh66ch548h556q"
            }
```
Update this hash to openstackdeploy.yaml file:
```
[root@dell-r640-009 deploy_default]# cat openstackdeploy.yaml
apiVersion: osp-director.openstack.org/v1beta1
kind: OpenStackDeploy
metadata:
  name: default
spec:
  configVersion: n67dh579hf8h65h545h66hb7h689h5c9h89hf4h648h79h5ch57h567hbfh77h5d6h658h559hcch579h5bh9fhb5h65fh55ch5bfh66ch548h556q
  configGenerator: default
```

6. Delete existing openstackdeploy and create a new one:
```
[root@dell-r640-009 deploy_default]# oc delete openstackdeploy default
[root@dell-r640-009 deploy_default]# oc create -f openstackdeploy.yaml
openstackdeploy.osp-director.openstack.org/default created
```

7. Mornitor deployment logs with command:
```
[root@dell-r640-009 deploy_default]# oc logs -f jobs/deploy-openstack-default
I0616 06:28:02.276426       1 deploy.go:323] Running deploy command.
...
PLAY RECAP *********************************************************************
compute-0                  : ok=346  changed=132  unreachable=0    failed=0    skipped=182  rescued=0    ignored=0   
compute-1                  : ok=343  changed=132  unreachable=0    failed=0    skipped=182  rescued=0    ignored=0   
controller-0               : ok=416  changed=167  unreachable=0    failed=0    skipped=188  rescued=0    ignored=0   
undercloud                 : ok=99   changed=26   unreachable=0    failed=0    skipped=12   rescued=0    ignored=1   
```

8.  Check configurations are applied to overcloud:
```
[root@dell-r640-009 deploy_default]# oc rsh openstackclient
sh-4.4$ cd 
sh-4.4$ cd work/n67dh579hf8h65h545h66hb7h689h5c9h89hf4h648h79h5ch57h567hbfh77h5d6h658h559hcch579h5bh9fhb5h65fh55ch5bfh66ch548h556q/
sh-4.4$ cd playbooks/tripleo-ansible/
sh-4.4$ grep parameter -r | grep admin
```
