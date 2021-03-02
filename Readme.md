# Velero + Restic integration with K8s and MinIO .

* Velero is an open source tool to safely backup and restore, perform disaster recovery, and migrate Kubernetes cluster resources and persistent volumes.
* Restic is a modern backup program that can back up your files from Linux, BSD, Mac and Windows, to many different storage types, including self-hosted and online services.
* We will use a bucket S3 from MinIO for Object Store, using AWS Velero provider, and MinIO for Volume Snapshotter using Restic.


## Index

1. Requirements
2. Install MinIO
3. Install Velero with Restic
4. Backup
5. Restore
6. source of information


## 1) Requirements:

* Kubernetes cluster, v1.10 or later.
* Storageclasses. You need one to create persistent volumes.

## 2) Install MinIO

You can deploy MinIO using official documentation from "https://docs.min.io/docs/deploy-minio-on-kubernetes.html", you can deploy using Helm or deploy the Operator.
We have deploy MinIO using Helm3 controlling the deploy with ArgoCD.
If you want to deploy MinIO using Argocd, fork or clone the repositorie "https://github.com/minio/charts", change the values according your requirements, and create the application from ArgoCD. Once you have your repositorie ready create the applitacion in ArgoCD.

Example:

```
argocd app create eks-tyc-minio --repo https://github.com/vass-engineering/EKS-TyC-minio.git --revision EKS-TyC-1.0.0 --path minio --dest-server https://kubernetes.default.svc --dest-namespace eks-tyc-minio --project default --values values.yaml --sync-policy automated --auto-prune --self-heal
```
<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/ArgoCD.png" width="700">


Once you have deployed MinIO in one way or the other, access to MinIO console. In order to access to the console you can use the ingress if you enabled to true the ingress parameters in the values.yaml, or you can access using port-fordward. "kubectl port-
forward <pod-name 9000", in the last case you should access as http://localhost:9000.

Obtain the  ACCESS_KEY and SECRET_KEY from the k8s secret created.

SECRET_KEY=$(kubectl get secret eks-tyc-minio -o jsonpath="{.data.secretkey}" | base64 --decode)
echo $SECRET_KEY
ACCESS_KEY=$(kubectl get secret eks-tyc-minio -o jsonpath="{.data.accesskey}" | base64 --decode)
echo $ACCESS_KEY


<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/MinioAccess.png" width="700">


Create a bucket and change the permisions :

<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/Createbucket.png" width="700">

<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/MinioPolicy.png" width="700">


## 3)Install Velero with Restic

* Create a file "credentials-velero" with the S3(Noobaa) credential access.

```
vi credentials-velero
```

```
[default]
aws_access_key_id = 8Vv12B5LXXXXXXXXXXXXXXX
aws_secret_access_key = m0scKSLvhv90XXXXXXXXXXXXXXXXXXXx
```


#### Install Velero

* Change <> with your environment details:

```
velero install --provider <provider>  --plugins <plugins,...>  --bucket <bucket-name>  --secret-file  <secret-file>   --use-restic --backup-location-config region=<region>,s3ForcePathStyle="true",s3Url=http://<S3-URL>:9000,publicUrl=https://<S3-URL> --use-volume-snapshots=false  --image velero/velero:v1.4.0
```

* Example:

```
velero install --provider aws --plugins velero/velero-plugin-for-aws:v1.0.1 --bucket backupeks --secret-file ./credentials-minio  --use-restic --backup-location-config region=minio,s3ForcePathStyle="true",s3Url=http://eks-tyc-minio.eks-tyc-minio.svc.cluster.local:9000,publicUrl=https://minio.tyc.vass.es --use-volume-snapshots=false  --image velero/velero:v1.4.0
```

## 4) Backup

* Backup a namespace

```
velero backup create <backup-name>  --snapshot-volumes=false  --include-namespaces  <namespace>
```

example:
```
velero backup create eks-tyc-kube-opex-analytics-20022021  --snapshot-volumes=false  --include-namespaces  eks-tyc-kube-opex-analytics
```

You can see you backup and details with the next velero commands, also you will see in MinIO  your new Backup.

```
velero get backups                                                                                                                    
NAME                                   STATUS      CREATED                         EXPIRES   STORAGE LOCATION   SELECTOR
eks-tyc-kube-opex-analytics-22022021   Completed   2021-02-22 11:21:16 +0100 CET   28d       default            <none>
```

```
velero describe backup eks-tyc-kube-opex-analytics-22022021                                                                            
Name:         eks-tyc-kube-opex-analytics-22022021
Namespace:    velero
Labels:       velero.io/storage-location=default
Annotations:  velero.io/source-cluster-k8s-gitversion=v1.18.9-eks-d1db3c
              velero.io/source-cluster-k8s-major-version=1
              velero.io/source-cluster-k8s-minor-version=18+

Phase:  Completed
```

<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/BackupMinio.png" width="700">




* Backup a NAMESPACE including PVS

You must add an annotation in order to backup a pv.

Annotations for Restic:

```
kubectl  annotate pod/<pod-name> backup.velero.io/backup-volumes=<volume-name>
```

```
kubectl  annotate pod/eks-tyc-kube-opex-analytics-0 backup.velero.io/backup-volumes=data-vol-eks
```

Backup the namespace including the pv.

```
velero backup create eks-tyc-kube-opex-analytics-20022021  --snapshot-volumes=false  --include-namespaces  eks-tyc-kube-opex-analytics
```

When you backup a PV using Restic, you will see in MinIO, in the bucket the snapshot. Restic will store the differents snapshots from one NameSpace in the folder Snapshots.  

<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/MinioRestic.png" width="700">
<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/MinioRestic1.png" width="700">
<img src="https://github.com/vass-engineering/Demo-backupK8s-Velero-Restic-Minio/blob/main/DocsImages/MinioResticSnapShots.png" width="700">


* Backup all the cluster.

```
velero backup create <name of the backup>
```

## 5) Restore

* List all the backups

```
velero get backups
```

* Manual Restore

```
velero restore create <RESTORE_NAME> --from-backup <BACKUP_NAME>
```

* Create a restore including the nginx and default namespaces

```
velero restore create --from-backup backup-1 --include-namespaces nginx,default
```

* Create a restore excluding the kube-system and default namespaces

```
velero restore create --from-backup backup-1 --exclude-namespaces kube-system,default
```


## 6) source of information

https://github.com/vmware-tanzu/velero
https://documentation.suse.com/suse-caasp/4.2/html/caasp-admin/_backup_and_restore_with_velero.html#_restore
https://velero.io/
https://www.digitalocean.com/community/tutorials/how-to-back-up-and-restore-a-kubernetes-cluster-on-digitalocean-using-velero
https://docs.pivotal.io/tkgi/1-10/velero-install.html
