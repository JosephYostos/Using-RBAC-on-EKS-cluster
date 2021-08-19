# Using-RBAC-on-EKS-cluster

## Create an IAM user


1- create a new user called David, and generate/save credentials for it:

```bash
    aws iam create-user --user-name David
    aws iam create-access-key --user-name David | tee /tmp/David_output.json
```

By running the previous step, you should get a response similar to:

```bash
{
    "AccessKey": {
        "UserName": "David",
        "Status": "Active",
        "CreateDate": "2021-07-28T15:37:27Z",
        "SecretAccessKey": < AWS Secret Access Key > ,
        "AccessKeyId": < AWS Access Key >
    }
}
```

2- To make it easy to switch back and forth between the admin user you created the cluster with, and this new user (David), run the following command to create a script that when sourced, sets the active user to be David:

```bash
cat << EoF > David_creds.sh
export AWS_SECRET_ACCESS_KEY=$(jq -r .AccessKey.SecretAccessKey /tmp/David_output.json)
export AWS_ACCESS_KEY_ID=$(jq -r .AccessKey.AccessKeyId /tmp/David_output.json)
EoF
```

## Map an IAM user To K8S

1- Run the following to get the existing ConfigMap and save into a file called aws-auth.yaml:

```bash
kubectl get configmap -n kube-system aws-auth -o yaml | grep -v "creationTimestamp\|resourceVersion\|selfLink\|uid" | sed '/^  annotations:/,+2 d' > aws-auth.yaml
```
2- append David user mapping to the existing configMap

```bash
cat << EoF >> aws-auth.yaml
data:
  mapUsers: |
    - userarn: arn:aws:iam::${ACCOUNT_ID}:user/David
      username: David
EoF
```

Note: to get the account ID login to IAM, click users, select the user David and copy the User ARN from there.

To verify everything populated and was created correctly, cat aws-auth.yaml and the output should be similar to the following:

```bash
apiVersion: v1
kind: ConfigMap
metadata:
  name: aws-auth
  namespace: kube-system
data:
  mapUsers: |
    - userarn: arn:aws:iam::123456789:user/David
      username: David
```

3- apply the ConfigMap to apply this mapping to the system:

```bash
kubectl apply -f aws-auth.yaml
```


## Test the new user

1- Issue the following command to source the David's AWS IAM user environmental variables:

```bash
. David_creds.sh
```
NOTE: for mac users you may need to use "source David_credssh" instead.

2- By running the above command, you’ve now set AWS environmental variables which should override the default admin user or role. To verify we’ve overrode the default user settings, run the following command:

```bash
aws sts get-caller-identity
```

output should be similar to:

```bash
{
    "Account": <AWS Account ID>,
    "UserId": <AWS User ID>,
    "Arn": "arn:aws:iam::<AWS Account ID>:user/David"
}
```

3- Run the following to unset the environmental variables that define us as David:

```bash
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
```

## Create a sample role/rolebinding to allow pods list

1- we’ll create a role called pod-reader that provides list, get, and watch access for pods and deployments, but only for the test namespace. Run the following to create this role:

```bash
cat << EoF > David-role.yaml
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
cat << EoF > David-role-binding.yaml
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: read-pods
  namespace: test
subjects:
- kind: User
  name: David
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

##Verify the role and Binding

1- Issue the following command that sources the David env vars, and verifies they’ve taken:

```bash
. David_creds.sh; aws sts get-caller-identity
```

2- As David, issue the following to get pods in the test namespace:

```bash
kubectl get pods -n test
```

Output should be similar to 

```bash
NAME                    READY     STATUS    RESTARTS   AGE
nginx-55bd7c9fd-kmbkf   1/1       Running   0          23h
```

If you try any other namespace you should get error similar to the following:

```bash
No resources found.
Error from server (Forbidden): pods is forbidden: User "rbac-user" cannot list resource "pods" in API group "" in the namespace "kube-system"
```


## Cleanup

- you can cleanup the files and resources you created by issuing the following commands

```bash
unset AWS_SECRET_ACCESS_KEY
unset AWS_ACCESS_KEY_ID
kubectl delete namespace test
rm David_creds.sh
rm David-role.yaml
rm David-role-binding.yaml
aws iam delete-access-key --user-name=David --access-key-id=$(jq -r .AccessKey.AccessKeyId /tmp/create_output.json)
aws iam delete-user --user-name David
rm /tmp/David_output.json
```
- Next remove the rbac-user mapping from the existing configMap by editing the existing aws-auth.yaml file:

```bash
data:
  mapUsers: |
    []
```

- And apply the ConfigMap and delete the aws-auth.yaml file

```bash
kubectl apply -f aws-auth.yaml
rm aws-auth.yaml
```


#### Referance: https://www.eksworkshop.com/beginner/090_rbac/intro/
