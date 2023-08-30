---
layout: post
title: How to deploy OpenShift ADP?
---

## Environment
- Red Hat Openshift Platform 4.12.27

## Prerequisites
- Deploy Openshift Data Fundation

## Steps 
1. [Install the OADP Operator](https://docs.openshift.com/container-platform/4.12/backup_and_restore/application_backup_and_restore/installing/installing-oadp-ocs.html)
2. Retrieve the Multicloud Object Gateway (MCG) credentials in order to create a Secret custom resource (CR) for the OpenShift API for Data Protection (OADP).
```
# oc get secret noobaa-admin -n openshift-storage -o json | jq .data
{
  "AWS_ACCESS_KEY_ID": "THpDTzBMNGg0bmJFb21ZakdDeWQ=",
  "AWS_SECRET_ACCESS_KEY": "UmZ4MlRtelFNWUIxeVBTSkc5OU1ia2Noa1lPVS9CMk9yRXR0aWRnNQ==",
  "email": "YWRtaW5Abm9vYmFhLmlv",
  "password": "ZllJNm40cnRCZkxwbmVZWVNBckI0QT09",
  "system": "bm9vYmFh"
}
```
3. Create a credentials-velero file
```
# cat << EOF > ./credentials-velero
[default]
aws_access_key_id=THpDTzBMNGg0bmJFb21ZakdDeWQ=
aws_secret_access_key=UmZ4MlRtelFNWUIxeVBTSkc5OU1ia2Noa1lPVS9CMk9yRXR0aWRnNQ==
EOF
```
4. Create a Secret with the default name
```
# oc create secret generic cloud-credentials -n openshift-adp --from-file cloud=credentials-velero
secret/cloud-credentials created
```
5. Installing the Data Protection Application
```
# cat oadp.yaml 
apiVersion: oadp.openshift.io/v1alpha1
kind: DataProtectionApplication
metadata:
  name: dpa-sample
  namespace: openshift-adp
spec:
  configuration:
    velero:
      defaultPlugins:
        - aws
        - openshift
      resourceTimeout: 10m
    restic:
      enable: true
      podConfig:
        nodeSelector:
          kubernetes.io/os: linux
  backupLocations:
    - velero:
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
          prefix: oadp
```
S3 URL:
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
6. Create DataProtectionApplication with command:
```
# oc create -f oadp.yaml 
dataprotectionapplication.oadp.openshift.io/dpa-sample created
```




