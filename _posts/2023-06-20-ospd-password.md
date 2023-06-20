---
layout: post
title: How to update overcloud passwords for existing environment with openstack director operator?
---

There are 2 secrets in `openstack` namespace stores overcloud passwords, `userpassword` and `tripleo-passwords`. 
`NodeRootPassword` is stored in secret `userpassword`, since it will be used by `cloud-init` during firstboot,  this cannot be updated for existing environment. 
We can follow procedures mentioned [here](https://access.redhat.com/documentation/en-us/red_hat_openstack_platform/16.2/html/rhosp_director_operator_for_openshift_container_platform/assembly_preparing-for-overcloud-deployment-with-the-director-operator_rhosp-director-operator#proc_setting-the-root-password-for-nodes_assembly_preparing-for-overcloud-deployment-with-the-director-operator) to specify `NodeRootPassword`.

And overcloud service passwords are stored in secret `tripleo-passwords`. 
We can follow below procedures to update those passwords for existing environment. 

1. Get and decode current passwords in secret `tripleo-passwords`:
```
[root@dell-r640-009 ~]# oc get secret tripleo-passwords -n openstack -o yaml 
apiVersion: v1
data:
  tripleo-overcloud-passwords.yaml: cGFyYW1ldGVyX2RlZmF1bHRzOgogIEFkbWluUGFzc3dvcmQ6IGNkZzduOXNzenJxN25iMm56enBsc3JsZzYKICBBZG1pblRva2VuOiA2aHdxcXB6c2RmcHRkcjdkejh0cDl3OWRtCiAgQW9kaFBhc3N3b3JkOiBuZGp3NHZzZ2YyN2MydmZ4MmtiOWh3azluCiAgQmFyYmljYW5QYXNzd29yZDogOWtyaHpwZ3RxYmM0bGc2eHc1NnZxZGw1YgogIEJhcmJpY2FuU2ltcGxlQ3J5cHRvS2VrOiBCd2tIQWdnR0F3SUVCQWdEQ1FVQkF3VUdDUVFEQndrSkJnZ0ZDQU1HQ1FrPQogIENlaWxvbWV0...
...
```
Decode the data:
```
[root@dell-r640-009 ~]# echo 'cGFyYW1ldGVyX2Rl...' | base64 -d > passwords.yaml
[root@dell-r640-009 ~]# cat passwords.yaml
parameter_defaults:
  ...
  MysqlRootPassword: testpassword  // change mysql root password
  NeutronPassword: testtest        // change neutron password in mysql database and keystone user neutron password
  ...
```
2. Update passwords in file `passwords.yaml` and encode it again with [online tool](https://www.base64encode.org/)
3. Confirm if `openstackdeploy` configVersion updated :
```
[root@dell-r640-009 deploy_default]# oc get -n openstack --sort-by {.metadata.creationTimestamp} osconfigversions -o json
...
host\n@@ -818 +864 @@   strategy: tri",
                "hash": "n67dh579hf8h65h545h66hb7h689h5c9h89hf4h648h79h5ch57h567hbfh77h5d6h658h559hcch579h5bh9fhb5h65fh55ch5bfh66ch548h556q"
            }
```
Comparing `configVersion` and if changed need to update in `openstackdeploy.yaml`
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
4. Recreate `openstackconfiggenerator` and `openstackdeploy` to apply this changes:
```
[root@dell-r640-009 yamls]# oc delete OpenStackConfigGenerator/default
openstackconfiggenerator.osp-director.openstack.org "default" deleted
[root@dell-r640-009 yamls]# oc create -f config_generator_default/openstackconfiggenerator.yaml 
openstackconfiggenerator.osp-director.openstack.org/default created
```
5. Monitor deployment logs:
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
6. Login to openstackclient pod and confirm those changes applied to ansible playbooks:
```
[root@dell-r640-009 ~]# oc rsh openstackclient
sh-4.4$ cd
sh-4.4$ cd work/n67dh579hf8h65h545h66hb7h689h5c9h89hf4h648h79h5ch57h567hbfh77h5d6h658h559hcch579h5bh9fhb5h65fh55ch5bfh66ch548h556q/
sh-4.4$ cd playbooks/tripleo-ansible/
sh-4.4$ grep password -r group_vars/ | grep neutron
group_vars/Controller:  neutron::db::mysql::password: testtest
group_vars/Controller:  neutron::keystone::authtoken::password: testtest
group_vars/Controller:  neutron::server::notifications::password: testtest
group_vars/Controller:  neutron::server::placement::auth_type: password
group_vars/Controller:  neutron::server::placement::password: testtest
group_vars/Controller:  nova::network::neutron::neutron_auth_type: v3password
group_vars/Controller:  nova::network::neutron::neutron_password: testtest
group_vars/Compute:  neutron::agents::ovn_metadata::auth_password: testtest
group_vars/Compute:  nova::network::neutron::neutron_auth_type: v3password
group_vars/Compute:  nova::network::neutron::neutron_password: testtest
sh-4.4$ grep password -r group_vars/ | grep root_password
group_vars/Controller:  mysql::server::root_password: testpassword
```
7. Login to controller and confirm the password changes applied:
```
[root@dell-r640-009 ~]# oc rsh openstackclient
sh-4.4$ ssh controller-0.ctlplane
[cloud-admin@controller-0 ~]$ sudo su -
[root@controller-0 ~]# podman ps | grep galera
0414538aa990  cluster.common.tag/rhosp16-openstack-mariadb:pcmklatest                                                  /bin/bash /usr/lo...  2 hours ago  Up 2 hours ago              galera-bundle-podman-0
[root@controller-0 ~]# podman exec -it 0414538aa990 bash
[root@controller-0 /]# mysql -u neutron -ptesttest
Welcome to the MariaDB monitor.  Commands end with ; or \g.
Your MariaDB connection id is 9403
Server version: 10.3.32-MariaDB MariaDB Server
MariaDB [(none)]>
```

