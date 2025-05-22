---
date: 2024-05-08
categories:
  - devops
  - aws
  - shell script
  - kubernetes
---
Amazon ECR Public allows users to store and access public container images. While ECR Public repositories are open to the public, access to pull or download images from these repositories may still require authentication.

While there are multiple reasons such as access control and security concern, the main benefit of getting an authentication token or login is to deal with rate limiting in my use case.

<!-- more -->

## Get a ECR-Public Token

In so doing, you can create a kubernetes secret from the token to be accessible within the namespace:

``` bash
TOKEN=$(aws ecr-pubic get-authorization-token --region us-east-1)
NS="THE_NAMESPACE"

kubectl create secret docker-registry ecr-public-secret \
  --namespace $NS \
  --docker-username=AWS
  --docker-password="$TOKEN" \
  --docker-server="public.erc.aws"
  --dry-run=client -o yaml | kubectl apply -f -

```

## Get a ECR-Public Login

Another option is to get the login password and then login to the ECR public registry. This option is more suitable for CI/CD automation script.

```bash
aws ecr-public get-login-password --region us-east-1 | docker login --username AWS --password-stdin public.ecr.aws
```

## Refresh Token/Login

The token/login has a validity period of 12 hours. You need to obtain a new one to continue accessing ECR Public.

The follow cron job will refresh the Kubernetes secret every 6 hours

``` bash
apiVersion: batch/v1beta1
kind: CronJob
metadata:
  name: refresh-ecr-public-token
spec:
  schedule: "0 */6 * * *"  # Runs every 6 hours
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: refresh-token
            image: awscli/awscli:latest  # Use an image with AWS CLI installed
            command: ["sh", "-c"]
            args:
            - |
              # Refresh ECR Public token
              token=$(aws ecr-public get-login-password --region us-east-1)
              kubectl create secret docker-registry ecr-public-secret \
                --docker-server=public.ecr.aws \
                --docker-username=AWS \
                --docker-password=$token
          restartPolicy: OnFailure
```
