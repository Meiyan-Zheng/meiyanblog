---
layout: post
title: How to deploy OpenShift ADP?
---

## Environment
- Red Hat Openshift Platform 4.12.27

## Prerequisites
- Deploy Openshift Data Fundation

## Steps 
1. [Install the OADP Operator](https://docs.openshift.com/container-platform/4.12/backup_and_restore/application_backup_and_restore/installing/installing-oadp-mcg.html)
2. Retrieve the Multicloud Object Gateway (MCG) credentials in order to create a Secret custom resource (CR) for the OpenShift API for Data Protection (OADP).
```
# oc get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d'
LzCO0L4h4nbEomYjGCyd
[root@10 ~]# oc get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d'
Rfx2TmzQMYB1yPSJG99MbkchkYOU/B2OrEttidg5
```
3. Create a credentials-velero file
```
# cat << EOF > ./credentials-velero
[default]
aws_access_key_id=LzCO0L4h4nbEomYjGCyd
aws_secret_access_key=Rfx2TmzQMYB1yPSJG99MbkchkYOU/B2OrEttidg5
EOF
```
4. Create a Secret with the default name
```
# oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=credentials-velero
secret/cloud-credentials created
```
5. Confirm S3 URL:
```
# oc describe noobaa -n openshift-storage | grep serviceS3 -A5
    serviceS3:
      External DNS:
        https://s3-openshift-storage.apps.ostest.test.metalkube.org
      Internal DNS:
        https://s3.openshift-storage.svc:443
      Internal IP:
        https://172.30.229.36:443
```
6. Create bucket with name `oadpbucket` 
```
# curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
# unzip awscliv2.zip
# sudo ./aws/install
# oc port-forward -n openshift-storage service/s3 10443:443 &
# NOOBAA_ACCESS_KEY=$(oc get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_ACCESS_KEY_ID|@base64d')
# NOOBAA_SECRET_KEY=$(oc get secret noobaa-admin -n openshift-storage -o json | jq -r '.data.AWS_SECRET_ACCESS_KEY|@base64d')
# alias s3='AWS_ACCESS_KEY_ID=$NOOBAA_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$NOOBAA_SECRET_KEY aws --endpoint https://localhost:10443 --no-verify-ssl s3'
# s3 ls
Handling connection for 10443
2023-08-30 08:04:14 first.bucket
# AWS_ACCESS_KEY_ID=$NOOBAA_ACCESS_KEY AWS_SECRET_ACCESS_KEY=$NOOBAA_SECRET_KEY aws s3api create-bucket --bucket oadpbucket --endpoint https://localhost:10443 --no-verify-ssl
# s3 ls
Handling connection for 10443
2023-08-30 08:04:14 oadpbucket
2023-08-30 08:04:14 first.bucket
```
7. Installing the Data Protection Application
```
# cat oadp.yaml 
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: velero-sample
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        \- openshift
        \- aws
      resourceTimeout: 10m
    restic:
      enable: true
      podConfig:
        nodeSelector:
          kubernetes.io/os: linux
  backupLocations:
    \- velero:
        config:
          profile: "default"
          region: minio
          s3Url: https://s3.openshift-storage.svc:443 
          insecureSkipTLSVerify: "true"
          s3ForcePathStyle: "true"
        provider: aws
        default: true
        credential:
          key: cloud
          name: cloud-credentials
        objectStorage:
          bucket: oadpbucket
          prefix: velero
```
8. Create DataProtectionApplication with command:
```
# oc create -f oadp.yaml 
dataprotectionapplication.oadp.openshift.io/dpa-sample created
```
9. Confirm the installation
```
# oc get dpa
NAME            AGE
velero-sample   6m52s
# oc get backupstoragelocation
NAME              PHASE       LAST VALIDATED   AGE    DEFAULT
velero-sample-1   Available   50s              7m3s   true
```



