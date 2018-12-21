---
title: CSI Driver with AWS
description: Vault CSI Driver with AWS secret engine
menu:
  product_vault:
    identifier: aws-csi-driver
    name: AWS CSI Driver
    parent: aws
    weight: 10
product_name: csi-driver
menu_name: product_vault
section_menu_id: guides
---
# Setup AWS secret engine for Vault CSI Driver

## Before you Begin

At first, you need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your cluster. If you do not already have a cluster, you can create one by using [Minikube](https://github.com/kubernetes/minikube).

Now, you need to have vault installed either on your cluster or outside the cluster. If you want to install Vault on your cluster, you can do it by running `kubectl apply -f ` to [this](/docs/examples/csi-driver/vault-install.yaml) file.

To keep things isolated, this tutorial uses a separate namespace called `demo` throughout this tutorial.

```console
$ kubectl create ns demo
namespace "demo" created

$ kubectl get ns demo
NAME    STATUS  AGE
demo    Active  5s
```

>Note: Yaml files used in this tutorial stored in [docs/examples/csi-driver/aws](/docs/examples/csi-driver/aws) folder in github repository [kubevault/docs](https://github.com/kubevault/docs)

## Configure AWS

Create IAM policy on AWS with following and copy the value of policy ARN:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "iam:AttachUserPolicy",
        "iam:CreateAccessKey",
        "iam:CreateUser",
        "iam:DeleteAccessKey",
        "iam:DeleteUser",
        "iam:DeleteUserPolicy",
        "iam:DetachUserPolicy",
        "iam:ListAccessKeys",
        "iam:ListAttachedUserPolicies",
        "iam:ListGroupsForUser",
        "iam:ListUserPolicies",
        "iam:PutUserPolicy",
        "iam:RemoveUserFromGroup"
      ],
      "Resource": [
        "arn:aws:iam::ACCOUNT-ID-WITHOUT-HYPHENS:user/vault-*"
      ]
    }
  ]
}
```

<p align="center">
  <img alt="AWS IAM Policy" src="/docs/images/csi-driver-policy-aws.jpg" style="padding: 10px;">
</p>

## Configure Vault

To use secret from `AWS` secret engine, you have to do following things.

1. **Enable `AWS` Engine:** To enable `AWS` secret engine run the following command.

   ```console
   $ vault secrets enable aws
   Success! Enabled the aws secrets engine at: aws/
   ```

2. **Create Engine Policy:**  To read secret from engine, we need to create a policy with `read` capability. Create a `policy.hcl` file and write the following content:

   ```yaml
   # capability of get secret
    path "aws/*" {
        capabilities = ["read"]
    }
   ```

    Write this policy into vault naming `test-policy` with following command:

    ```console
    $ vault policy write test-policy policy.hcl
    Success! Uploaded policy: test-policy
    ```

3. **Crete AWS config:** To communicate with AWS for generating IAM credentials, Vault needs to configure credentials. Run:

    ```console
    $ vault write aws/config/root \
      access_key=AKIAJWVN5Z4FOFT7NLNA \
      secret_key=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 \
      region=us-east-1
    Success! Data written to: aws/config/root  
    ```

4. **Configure a Vault Role:** We need to configure a vault role that maps to a set of permissions in AWS and an AWS credential type. When users generate credentials, they are generated against this role,

    ```console
    $ vault write aws/roles/my-aws-role \
      arn=arn:aws:iam::452618475015:policy/vaultiampolicy \ # In AWS configuration ACCOUNT-ID-WITHOUT-HYPHENS = vaultiampolicy
      credential_type=iam_user \
      policy_document=-<<EOF
    {
      "Version": "2012-10-17",
      "Statement": [
        {
          "Effect": "Allow",
          "Action": "ec2:*",
          "Resource": "*"
        }
      ]
    }
    EOF
    Success! Data written to: aws/roles/my-aws-role
    ```

   Here, `my-aws-role` will be treated as secret name on storage class.

## Configure Cluster

1. **Create Service Account:** Create `service.yaml` file with following content:

    ```yaml
    apiVersion: rbac.authorization.k8s.io/v1beta1
    kind: ClusterRoleBinding
    metadata:
      name: role-awscreds-binding
      namespace: demo
    roleRef:
      apiGroup: rbac.authorization.k8s.io
      kind: ClusterRole
      name: system:auth-delegator
    subjects:
    - kind: ServiceAccount
      name: aws-vault
      namespace: demo
    ---
    apiVersion: v1
    kind: ServiceAccount
    metadata:
      name: aws-vault
      namespace: demo
    ```
   After that, run `kubectl apply -f service.yaml` to create a service account.

2. **Enable Kubernetes Auth:**  To enable Kubernetes auth backend, we need to extract the token reviewer JWT, Kubernetes CA certificate and Kubernetes host information.

    ```console
    export VAULT_SA_NAME=$(kubectl get sa aws-vault -n demo -o jsonpath="{.secrets[*]['name']}")

    export SA_JWT_TOKEN=$(kubectl get secret $VAULT_SA_NAME -n demo -o jsonpath="{.data.token}" | base64 --decode; echo)

    export SA_CA_CRT=$(kubectl get secret $VAULT_SA_NAME -n demo -o jsonpath="{.data['ca\.crt']}" | base64 --decode; echo)

    export K8S_HOST=<host-ip>
    export K8s_PORT=6443
    ```

    Now, we can enable the Kubernetes authentication backend and create a Vault named role that is attached to this service account. Run:

    ```console
    $ vault auth enable kubernetes
    Success! Enabled kubernetes auth method at: kubernetes/

    $ vault write auth/kubernetes/config \
        token_reviewer_jwt="$SA_JWT_TOKEN" \
        kubernetes_host="https://$K8S_HOST:$K8s_PORT" \
        kubernetes_ca_cert="$SA_CA_CRT"
    Success! Data written to: auth/kubernetes/config

    $ vault write auth/kubernetes/role/aws-cred-role \
        bound_service_account_names=aws-vault \
        bound_service_account_namespaces=demo \
        policies=test-policy \
        ttl=24h
    Success! Data written to: auth/kubernetes/role/aws-cred-role
    ```

    Here, `aws-cred-role` is the name of the role.

3. **Create AppBinding:** To connect CSI driver with Vault, we need to create an `AppBinding`. First we need to make sure, if `AppBinding` CRD is installed in your cluster by running:

    ```console
    $ kubectl get crd -l app=catalog
    NAME                                          CREATED AT
    appbindings.appcatalog.appscode.com           2018-12-12T06:09:34Z
    ```

    If you don't see that CRD, then you can partially install this with following command, otherwise skip this command

    ```console
    kubectl apply -f https://raw.githubusercontent.com/kmodules/custom-resources/master/api/crds/appbinding.yaml

    ```

    If AppBinding CRD is installed, Create AppBinding with the following data:

    ```yaml
    apiVersion: appcatalog.appscode.com/v1alpha1
    kind: AppBinding
    metadata:
      name: vaultapp
      namespace: demo
    spec:
    clientConfig:
      url: http://165.227.190.238:30001 # Replace this with Vault URL
      insecureSkipTLSVerify: true
    parameters:
      apiVersion: "kubevault.com/v1alpha1"
      kind: "VaultServerConfiguration"
      usePodServiceAccountForCSIDriver: true
      authPath: "kubernetes"
      policyControllerRole: aws-cred-role # we created this in previous step
    ```

4. **Create StorageClass:** Create `storage-class.yaml` file with following content, then run `kubectl apply -f storage-class.yaml`

    ```yaml
    kind: StorageClass
    apiVersion: storage.k8s.io/v1
    metadata:
      name: vault-aws-storage
      namespace: demo
    annotations:
      storageclass.kubernetes.io/is-default-class: "false"
    provisioner: secrets.csi.kubevault.com # For Kubernetes 1.12.x(csi-vault:0.1.0) use -> com.kubevault.csi.secrets
    parameters:
      ref: demo/vaultapp # namespace/AppBinding, we created this in previous step
      engine: AWS # vault engine name
      role: my-aws-role # role name on vault which you want get access
      path: aws # specify the secret engine path, default is aws
    ```
   > N.B: If you use csi-vault:0.1.0, use `com.kubevault.csi.secrets` as provisioner name.

## Test & Verify

1. **Create PVC:** Create a `PersistantVolumeClaim` with following data. This makes sure a volume will be created and provisioned on your behalf.

    ```yaml
    apiVersion: v1
    kind: PersistentVolumeClaim
    metadata:
      name: csi-pvc
      namespace: demo
    spec:
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 1Gi
      storageClassName: vault-aws-storage
      volumeMode: DirectoryOrCreate
    ```

2. **Create Pod:** Now we can create a Pod which refers to this volume. When the Pod is created, the volume will be attached, formatted and mounted to the specific container.

    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: mypod
      namespace: demo
    spec:
      containers:
      - name: mypod
        image: busybox
        command:
          - sleep
          - "3600"
        volumeMounts:
        - name: my-vault-volume
          mountPath: "/etc/foo"
          readOnly: true
      serviceAccountName: aws-vault
      volumes:
        - name: my-vault-volume
          persistentVolumeClaim:
            claimName: csi-pvc
    ```

   Check if the Pod is running successfully, by running:

    ```console
    kubectl describe pods/my-pod
    ```

3. **Verify Secret:** If the Pod is running successfully, then check inside the app container by running

    ```console
    $ kubectl exec -ti mypod /bin/sh -n demo
    / # ls /etc/foo
    access_key  secret_key
    / # cat /etc/foo/access_key
    AKIAIH4QGZQOCMIWYLDA
    ```

   So, we can see that the aws IAM credentials `access_key` and  `secret_key` are mounted into the pod

## Cleaning up

To cleanup the Kubernetes resources created by this tutorial, run:

```console
$ kubectl delete ns demo
namespace "demo" deleted
```