# Using-RBAC-on-EKS-cluster

## Create a user

    ```bash
aws iam create-user --user-name rbac-user
aws iam create-access-key --user-name rbac-user | tee /tmp/create_output.json
    ```
