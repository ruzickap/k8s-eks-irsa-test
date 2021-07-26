# IRSA test

Usecase:

* Create IAM Role + policies before the cluster [requirement]
* Create EKS cluster
* Create IRSA with the pre-created IAM role
* Test if IRSA is working with pod accessing S3
* Change IAM Role policy (by adding new resource) to allow pod to access
  different S3 bucket

Create IAM Role with Policy allowing access to new S3 bucket:

```bash
export AWS_DEFAULT_REGION="eu-west-1"
cat > "/tmp/aws-s3-iam.yml" << \EOF
AWSTemplateFormatVersion: 2010-09-09
Description: >-
  IAM role for serviceaccount "s3-test/s3-test"
Resources:
  S3Policy:
    Type: AWS::IAM::Policy
    Properties:
      PolicyName: !Sub "${AWS::StackName}-AmazonS3"
      Roles:
        - !Ref Role1
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: Allow
          Action:
          - "s3:*"
          Resource:
          - !Sub "arn:aws:s3:::${AWS::StackName}"
          - !Sub "arn:aws:s3:::${AWS::StackName}/*"
  S3Bucket:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: Private
      BucketName: !Sub "${AWS::StackName}"
  S3Bucket2:
    Type: AWS::S3::Bucket
    Properties:
      AccessControl: Private
      BucketName: !Sub "${AWS::StackName}2"
  Role1:
    Type: 'AWS::IAM::Role'
    Properties:
      RoleName: !Sub "${AWS::StackName}"
      AssumeRolePolicyDocument:
        Statement:
          - Action:
              - 'sts:AssumeRoleWithWebIdentity'
            Effect: Allow
            Principal:
              Service:
              - eks.amazonaws.com
        Version: 2012-10-17
Outputs:
  Role1:
    Value: !GetAtt Role1.Arn
EOF
```

Initiate CloudFormation:

```bash
eval aws cloudformation deploy --capabilities CAPABILITY_NAMED_IAM \
  --stack-name "${USER}-s3-iam-test" --template-file "/tmp/aws-s3-iam.yml"
```

Get the IAM Role ARN (it will be used later):

```bash
S3_IAM_ARN=$(aws cloudformation describe-stacks --stack-name "${USER}-s3-iam-test" | jq -r ".Stacks[0].Outputs[] | select(.OutputKey==\"Role1\") .OutputValue")
echo "${S3_IAM_ARN}"
```

Create EKS cluster:

```bash
eksctl create cluster \
  --name=${USER}-k8s \
  --tags "Owner=${USER},Environment=Test,Division=Services" \
  --region=eu-west-1 \
  --managed \
  --nodegroup-name ${USER}-k8s \
  --node-type=t3.medium \
  --node-volume-size=30 \
  --with-oidc \
  --node-labels "Owner=${USER},Environment=Test,Division=Services" \
  --kubeconfig=/tmp/kubeconfig.conf
```

Check IAM role 1:

```bash
aws iam list-roles --query "Roles[?contains(RoleName, \`${USER}-s3-iam-test\`) == \`true\`]"
```

Create IRSA with previously created IAM role.
Details taken from: [Introducing fine-grained IAM roles for service accounts](https://aws.amazon.com/blogs/opensource/introducing-fine-grained-iam-roles-service-accounts/)

> `eksctl` doesn't configure/set the [trust relationship policy document](https://docs.aws.amazon.com/eks/latest/userguide/iam-roles-for-service-accounts-technical-overview.html#iam-role-configuration).

```bash
eksctl create iamserviceaccount --cluster ${USER}-k8s --attach-role-arn "${S3_IAM_ARN}" --namespace s3-test --name s3-test --approve --override-existing-serviceaccounts

ISSUER_URL=$(aws eks describe-cluster --name "${USER}-k8s" --query cluster.identity.oidc.issuer --output text)
ISSUER_HOSTPATH=$(echo $ISSUER_URL | cut -f 3- -d'/')
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
PROVIDER_ARN="arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${ISSUER_HOSTPATH}"

cat > /tmp/irp-trust-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "$PROVIDER_ARN"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${ISSUER_HOSTPATH}:sub": "system:serviceaccount:s3-test:s3-test"
        }
      }
    }
  ]
}
EOF

aws iam update-assume-role-policy --role-name "${USER}-s3-iam-test" --policy-document file:///tmp/irp-trust-policy.json
```

Check IAM role 2:

```bash
aws iam list-roles --query "Roles[?contains(RoleName, \`${USER}-s3-iam-test\`) == \`true\`]"
```

You should see something like:

```json
...
            "Statement": [
                {
                    "Effect": "Allow",
                    "Principal": {
                        "Federated": "arn:aws:iam::7xxxxxxxxxx7:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/id/Bxxxxxxxxxxxxxxxxxxxxxxxxx3"
                    },
                    "Action": "sts:AssumeRoleWithWebIdentity",
                    "Condition": {
                        "StringEquals": {
                            "oidc.eks.eu-west-1.amazonaws.com/id/Bxxxxxxxxxxxxxxxxxxxxxxxxx3:sub": "system:serviceaccount:s3-test:s3-test",
                        }
                    }
                }
...
```

Test IRSA - new pod which is using `s3-test` service account should be able to write to S3:

```bash
export KUBECONFIG="/tmp/kubeconfig.conf"
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: s3-test
  namespace: s3-test
spec:
  serviceAccountName: s3-test
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    env:
    - name: HOME
      value: "/tmp"
    - name: AWS_DEFAULT_REGION
      value: "${AWS_DEFAULT_REGION}"
    command:
      - /bin/bash
      - -c
      - |
        set -x
        aws s3 ls s3://${USER}-s3-iam-test/
        aws s3 cp /etc/hostname s3://${USER}-s3-iam-test/
        aws s3 ls s3://${USER}-s3-iam-test/
        aws s3 rm s3://${USER}-s3-iam-test/hostname
        echo "This should not work:"
        aws s3 cp /etc/hostname s3://${USER}-s3-iam-test2/
  restartPolicy: Never
EOF
```

Check the result of the operation by looking at the logs outputs:

```bash
sleep 10
kubectl logs -n s3-test s3-test
```

Add a new bucket to the existing CloudFormation stack, where I would like to
write some data from the pod:

```bash
sed -i '/arn:aws:s3:::${AWS::StackName}\/\*/a \ \ \ \ \ \ \ \ \ \ - !Sub "arn:aws:s3:::${AWS::StackName}2/*"\n \ \ \ \ \ \ \ \ \ - !Sub "arn:aws:s3:::${AWS::StackName}2"' /tmp/aws-s3-iam.yml
```

See the updated CloudFormation template:

```bash
cat /tmp/aws-s3-iam.yml
...
      PolicyDocument:
        Version: "2012-10-17"
        Statement:
        - Effect: Allow
          Action:
          - s3:ListBucket
          - s3:GetBucketLocation
          - s3:ListBucketMultipartUploads
          Resource: !GetAtt S3Bucket.Arn
        - Effect: Allow
          Action:
          - s3:PutObject
          - s3:GetObject
          - s3:DeleteObject
          - s3:ListMultipartUploadParts
          - s3:AbortMultipartUpload
          Resource:
          - !Sub "arn:aws:s3:::${AWS::StackName}"
          - !Sub "arn:aws:s3:::${AWS::StackName}/*"
          - !Sub "arn:aws:s3:::${AWS::StackName}2/*"
          - !Sub "arn:aws:s3:::${AWS::StackName}2"
...
```

Apply the template to existing CloudFormation stack:

```bash
aws cloudformation update-stack --stack-name "${USER}-s3-iam-test" --capabilities CAPABILITY_NAMED_IAM --template-body file:///tmp/aws-s3-iam.yml
```

Test IRSA - another pod which is using `s3-test` service account should be able to write
to another S3 bucket `${USER}-s3-iam-test2`

```bash
export KUBECONFIG="/tmp/kubeconfig.conf"
kubectl apply -f - << EOF
apiVersion: v1
kind: Pod
metadata:
  name: s3-test2
  namespace: s3-test
spec:
  serviceAccountName: s3-test
  containers:
  - name: aws-cli
    image: amazon/aws-cli
    env:
    - name: HOME
      value: "/tmp"
    - name: AWS_DEFAULT_REGION
      value: "${AWS_DEFAULT_REGION}"
    command:
      - /bin/bash
      - -c
      - |
        set -x
        export HOME=/tmp
        aws s3 ls s3://${USER}-s3-iam-test2/
        aws s3 cp /etc/hostname s3://${USER}-s3-iam-test2/
        aws s3 ls s3://${USER}-s3-iam-test2/
        aws s3 rm s3://${USER}-s3-iam-test2/hostname
        aws s3 cp /etc/hostname s3://${USER}-s3-iam-test/
        aws s3 rm s3://${USER}-s3-iam-test/hostname
  restartPolicy: Never
EOF
```

Check the result of the operation by looking at the logs outputs:

```bash
sleep 10
kubectl logs -n s3-test s3-test2
```
