# Using-RBAC-on-EKS-cluster

## Create an IAM user


1- create a new user called Peter, and generate/save credentials for it:

```bash
    aws iam create-user --user-name Peter
    aws iam create-access-key --user-name Peter | tee /tmp/Peter_output.json
```

By running the previous step, you should get a response similar to:

```bash
{
    "AccessKey": {
        "UserName": "Peter",
        "Status": "Active",
        "CreateDate": "2021-07-28T15:37:27Z",
        "SecretAccessKey": < AWS Secret Access Key > ,
        "AccessKeyId": < AWS Access Key >
    }
}
```

2- To make it easy to switch back and forth between the admin user you created the cluster with, and this new user (Peter), run the following command to create a script that when sourced, sets the active user to be Peter:

```bash
cat << EoF > Peter_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/Peter_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/Peter_output.json)
EoF
```

## Map an IAM user To K8S

1- Run the following to get the existing ConfigMap and save into a file called aws-auth.yaml:

```bash
kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml
```
2- append peter user mapping to the existing configMap

```bash
cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/Peter
      username: Peter
EoF
```

Note: to get the account ID login to IAM, click users, select the user peter and copy the User ARN from there.

To verify everything populated and was created correctly, cat aws-auth.yaml and the output should be similar to the following:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/Peter
      username: Peter
```

3- apply the ConfigMap to apply this mapping to the system:

```bash
kubectl apply -f aws-auth.yaml
```


## Test the new user

1- Issue the following command to source the Peter's AWS IAM user environmental variables:

```bash
. Peter_creds.sh
```

2- By running the above command, you’ve now set AWS environmental variables which should override the default admin user or role. To verify we’ve overrode the default user settings, run the following command:

```bash
aws sts get-caller-identity
```

output should be similar to:

```bash
{
    "Account": <AWS Account ID>,
    "UserId": <AWS User ID>,
    "Arn": "arn:aws:iam::<AWS Account ID>:user/Peter"
}
```

3- Run the following to unset the environmental variables that define us as Peter:

```bash
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
```

## Create a sample role/rolebinding to allow pods list

1- we’ll create a role called pod-reader that provides list, get, and watch access for pods and deployments, but only for the test namespace. Run the following to create this role:

```bash
cat << EoF > Peter-role.yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  namespace: test
  name: pod-reader
rules:
- apiGroups: [""] # "" indicates the core API group
  resources: ["pods"]
  verbs: ["list","get","watch"]
- apiGroups: ["extensions","apps"]
  resources: ["deployments"]
  verbs: ["get", "list", "watch"]
EoF
```

2- create rolebinding

```bash
cat << EoF > Peter-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: test
subjects:
- kind: User
  name: Peter
  apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: pod-reader
  apiGroup: rbac.authorization.k8s.io
EoF
```

3- create the role and rolebinding

```bash
kubectl apply -f rbacuser-role.yaml
kubectl apply -f rbacuser-role-binding.yaml
```



