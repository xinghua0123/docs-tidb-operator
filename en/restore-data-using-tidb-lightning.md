---
title: Import Data
summary: Learn how to quickly import data with TiDB Lightning.
aliases: ['/docs/tidb-in-kubernetes/dev/restore-data-using-tidb-lightning/']
---

# Import Data

This document describes how to import data into a TiDB cluster in Kubernetes using [TiDB Lightning](https://github.com/pingcap/tidb-lightning).

TiDB Lightning contains two components: tidb-lightning and tikv-importer. In Kubernetes, the tikv-importer is inside the separate Helm chart of the TiDB cluster. And tikv-importer is deployed as a `StatefulSet` with `replicas=1` while tidb-lightning is in a separate Helm chart and deployed as a `Job`.

TiDB Lightning supports [three backends](https://docs.pingcap.com/tidb/stable/tidb-lightning-backends): `importer`, `local`, and `tidb`.

- For the `importer` backend, both tikv-importer and tidb-lightning need to be deployed.
- For the `local` or `tidb` backend, only tidb-lightning needs to be deployed.
- For the `tidb` backend, it is recommended to import data using CustomResourceDefinition (CRD) in TiDB Operator v1.1 and later versions. For details, refer to [Restore Data from GCS Using TiDB Lightning](restore-from-gcs.md) or [Restore Data from S3-Compatible Storage Using TiDB Lightning](restore-from-s3.md)

## Deploy TiKV Importer

> **Note:**
>
> If you use the `local` or `tidb` backend for data restoration, you can skip deploying tikv-importer and [deploy tidb-lightning](#deploy-tidb-lightning) directly.

You can deploy tikv-importer using the Helm chart. See the following example:

1. Make sure that the PingCAP Helm repository is up to date:

    {{< copyable "shell-regular" >}}

    ```shell
    helm repo update
    ```

    {{< copyable "shell-regular" >}}

    ```shell
    helm search tikv-importer -l
    ```

2. Get the default `values.yaml` file for easier customization:

    {{< copyable "shell-regular" >}}

    ```shell
    helm inspect values pingcap/tikv-importer --version=${chart_version} > values.yaml
    ```

3. Modify the `values.yaml` file to specify the target TiDB cluster. See the following example:

    {{< copyable "" >}}

    ```yaml
    clusterName: demo
    image: pingcap/tidb-lightning:v4.0.9
    imagePullPolicy: IfNotPresent
    storageClassName: local-storage
    storage: 20Gi
    pushgatewayImage: prom/pushgateway:v0.3.1
    pushgatewayImagePullPolicy: IfNotPresent
    config: |
      log-level = "info"
      [metric]
      job = "tikv-importer"
      interval = "15s"
      address = "localhost:9091"
    ```

    `clusterName` must match the target TiDB cluster.

4. Deploy tikv-importer:

    {{< copyable "shell-regular" >}}

    ```shell
    helm install pingcap/tikv-importer --name=${cluster_name} --namespace=${namespace} --version=${chart_version} -f values.yaml
    ```

    > **Note:**
    >
    > You must deploy tikv-importer in the same namespace where the target TiDB cluster is deployed.

## Deploy TiDB Lightning

### Configure TiDB Lightning

Use the following command to get the default configuration of TiDB Lightning:

{{< copyable "shell-regular" >}}

```shell
helm inspect values pingcap/tidb-lightning --version=${chart_version} > tidb-lightning-values.yaml
```

Configure a `backend` used by TiDB Lightning according to your needs. The options include `importer`, `local`, and `tidb`.

> **Note:**
>
> If you use the [`local` backend](https://docs.pingcap.com/tidb/stable/tidb-lightning-backends#tidb-lightning-local-backend), you must set `sortedKV` in `values.yaml` to create the corresponding PVC. The PVC is used for local KV sorting.

TiDB Lightning Helm chart supports both local and remote data sources.

#### Local

In the local mode, the backup data must be on one of the Kubernetes node. To enable this mode, set `dataSource.local.nodeName` to the node name and `dataSource.local.hostPath` to the path of the backup data. The path should contain a file named `metadata`.

#### Remote

Unlike the local mode, the remote mode needs to use [rclone](https://rclone.org) to download the backup tarball file from a network storage to a PV. Any cloud storage supported by rclone should work, but currently only the following have been tested: [Google Cloud Storage (GCS)](https://cloud.google.com/storage/), [Amazon S3](https://aws.amazon.com/s3/), [Ceph Object Storage](https://ceph.com/ceph-storage/object-storage/).

To restore backup data from the remote source, take the following steps:

1. Make sure that `dataSource.local.nodeName` and `dataSource.local.hostPath` in `values.yaml` are commented out.

2. Grant permissions to remote storage access

    Like restoring data using BR and Dumpling, when using Amazon S3 as the storage, there are three methods to grant permissions. The configuration varies with different methods. For details, see [Back up the TiDB Cluster on AWS using BR](backup-to-aws-s3-using-br.md#three-methods-to-grant-aws-account-permissions). If you use Ceph or GCS as the storage, you can only grant permissions by importing AccessKey and SecretKey.

    * Grant permissions by importing AccessKey and SecretKey

        If you use Amazon S3, Ceph, or GCS as the storage, grant permissions by importing AccessKey and SecretKey.

        1. Create a `Secret` configuration file `secret.yaml` containing the rclone configuration. A sample configuration is listed below. Only one cloud storage configuration is required.

            {{< copyable "" >}}

            ```yaml
            apiVersion: v1
            kind: Secret
            metadata:
              name: cloud-storage-secret
            type: Opaque
            stringData:
              rclone.conf: |
                [s3]
                type = s3
                provider = AWS
                env_auth = false
                access_key_id = ${access_key}
                secret_access_key = ${secret_key}
                region = us-east-1
            
                [ceph]
                type = s3
                provider = Ceph
                env_auth = false
                access_key_id = ${access_key}
                secret_access_key = ${secret_key}
                endpoint = ${endpoint}
                region = :default-placement
            
                [gcs]
                type = google cloud storage
                # The service account must include Storage Object Viewer role
                # The content can be retrieved by `cat ${service-account-file} | jq -c .`
                service_account_credentials = ${service_account_json_file_content}
            ```

        2. Execute the following command to create `Secret`:

            {{< copyable "shell-regular" >}}

            ```shell
            kubectl apply -f secret.yaml -n ${namespace}
            ```

    * Grant permissions by associating IAM with Pod or with ServiceAccount

       If you use Amazon S3 as the storage, you can grant permissions by associating IAM with Pod or with ServiceAccount, in which `s3.access_key_id` and `s3.secret_access_key` can be ignored. 
        
        1. Save the following configurations as `secret.yaml`.
        
            {{< copyable "" >}}

            ```yaml
            apiVersion: v1
            kind: Secret
            metadata:
              name: cloud-storage-secret
            type: Opaque
            stringData:
              rclone.conf: |
                [s3]
                type = s3
                provider = AWS
                env_auth = true
                access_key_id =
                secret_access_key =
                region = us-east-1
            ```
            
        2. Execute the following command to create `Secret`:
        
            {{< copyable "shell-regular" >}}
        
            ```shell
            kubectl apply -f secret.yaml -n ${namespace}
            ```

3. Configure the `dataSource.remote.storageClassName` to an existing storage class in the Kubernetes cluster.

#### Ad hoc 

When restoring data from remote storage, sometimes the restore process is interrupted due to the exception. In such cases, if you do not want to download backup data from the network storage repeatedly, you can use the ad hoc mode to directly recover the data that has been downloaded and decompressed into PV in the remote mode. The steps are as follows:

1. Ensure `dataSource.local` and `dataSource.remote` in the config file `values.yaml` are empty。

2. Configure `dataSource.adhoc.pvcName` in `values.yaml` to the PVC name used in restoring data from remote storage.

3. Configure `dataSource.adhoc.backupName` in `values.yaml` to the name of the original backup data, such as: `backup-2020-12-17T10:12:51Z` (Do not contain the '. tgz' suffix of the compressed file name on network storage).

### Deploy TiDB Lightning

The method of deploying TiDB Lightning varies with different methods of granting permissions and with different storages.

* If you grant permissions by importing Amazon S3 AccessKey and SecretKey, or if you use Ceph or GCS as the storage, run the following command to deploy TiDB Lightning:

    {{< copyable "shell-regular" >}}

    ```shell
    helm install pingcap/tidb-lightning --name=${release_name} --namespace=${namespace} --set failFast=true -f tidb-lightning-values.yaml --version=${chart_version}
    ```

* If you grant permissions by associating Amazon S3 IAM with Pod, take the following steps:

    1. Create the IAM role:

        [Create an IAM role](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) for the account, and [grant the required permission](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_manage-attach-detach.html) to the role. The IAM role requires the `AmazonS3FullAccess` permission because TiDB Lightning needs to access Amazon S3 storage.

    2. Modify `tidb-lightning-values.yaml`, and add the `iam.amazonaws.com/role: arn:aws:iam::123456789012:role/user` annotation in the `annotations` field.

    3. Deploy TiDB Lightning:

        {{< copyable "shell-regular" >}}

        ```shell
        helm install pingcap/tidb-lightning --name=${release_name} --namespace=${namespace} --set failFast=true -f tidb-lightning-values.yaml --version=${chart_version}
        ```

        > **Note:**
        >
        > `arn:aws:iam::123456789012:role/user` is the IAM role created in Step 1.

* If you grant permissions by associating Amazon S3 with ServiceAccount, take the following steps:

    1. Enable the IAM role for the service account on the cluster:

        To enable the IAM role permission on the EKS cluster, see [AWS Documentation](https://docs.aws.amazon.com/eks/latest/userguide/enable-iam-roles-for-service-accounts.html).

    2. Create the IAM role:

        [Create an IAM role](https://docs.aws.amazon.com/eks/latest/userguide/create-service-account-iam-policy-and-role.html). Grant the `AmazonS3FullAccess` permission to the role, and edit `Trust relationships` of the role.

    3. Associate IAM with the ServiceAccount resources:

        {{< copyable "shell-regular" >}}

        ```shell
        kubectl annotate sa ${servieaccount} -n eks.amazonaws.com/role-arn=arn:aws:iam::123456789012:role/user
        ```

    4. Deploy TiDB Lightning:

        {{< copyable "shell-regular" >}}

        ```shell
        helm install pingcap/tidb-lightning --name=${release_name} --namespace=${namespace} --set-string failFast=true,serviceAccount=${servieaccount} -f tidb-lightning-values.yaml --version=${chart_version}
        ```

        > **Note:**
        >
        > `arn:aws:iam::123456789012:role/user` is the IAM role created in Step 1.
        > `${service-account}` is the ServiceAccount used by TiDB Lightning. The default value is `default`.

When TiDB Lightning fails to restore data, you cannot simply restart it. **Manual intervention** is required. So the TiDB Lightning's `Job` restart policy is set to `Never`.

If the lightning fails to restore data, follow the steps below to do manual intervention:

1. Delete the lightning job by running `kubectl delete job -n ${namespace} ${release_name}-tidb-lightning`.

2. Create the lightning job again with `failFast` disabled by `helm template pingcap/tidb-lightning --name ${release_name} --set failFast=false -f tidb-lightning-values.yaml | kubectl apply -n ${namespace} -f -`.

3. When the lightning pod is running again, use `kubectl exec -it -n ${namespace} ${pod_name} sh` to `exec` into the lightning container.

4. Get the startup script by running `cat /proc/1/cmdline`.

5. Diagnose the lightning following the [troubleshooting guide](https://pingcap.com/docs/stable/troubleshoot-tidb-lightning/).

## Destroy TiKV Importer and TiDB Lightning

Currently, TiDB Lightning can only restore data offline. When the restoration finishes and the TiDB cluster needs to provide service for applications, the TiDB Lightning should be deleted to save cost.

* To delete tikv-importer, run `helm delete ${release_name} --purge`.

* To delete tidb-lightning, run `helm delete ${release_name} --purge`.
