# lab-crossplane
The main objective would be to study and experiment with crossplane as well as use the localstack aws integrated with kubernetes provisioned via kind.

# Overview
* Install the kubernetes from kind
* Install aws localstack and expose services
* Install crossplane within kind kubernetes cluster
* Create a crossplane AWS provider integrated to the AWS localstack.
* Create kubernetes manifests to use crossplane and from there create the resources within aws localstack. (Like s3 bucket)  

## step 1  
**Install Kind**
```
curl -Lo ./kind https://kind.sigs.k8s.io/dl/latest/kind-linux-amd64
chmod +x ./kind
sudo mv ./kind /usr/local/bin/kind
```

## step 2
**Create cluster kind**
```
kind create cluster --name crossplane-lab
```

## step 3
**Install Localstack**
```
docker run -p 4566:4566 -p 4571:4571 localstack/localstack
```

***OR***

```
docker-compose up -d localstack
```

## step 4
**Check if localstack is running**
```
curl http://localhost:4566/health
```

# step 5
**Install the crossplane**
>> Here we need to install via Helm v3
```
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
```

**add the crossplane helm repo**
```
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
```

**Run helm install for crossplane**
```
helm install crossplane --namespace crossplane-system --create-namespace crossplane-stable/crossplane
```

**Install the AWS provider to crossplane**
```
cat >provider-aws-s3.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws-s3
  namespace: crossplane-system
spec:
  package: xpkg.upbound.io/upbound/provider-aws-s3:v0.40.0
EOF

kubectl apply -f provider-aws-s3.yaml
```

**create aws localstack secrets type**
```
cat >localstack-aws-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: localstack-aws-secret
  namespace: crossplane-system
stringData:
  creds: |
    [default]
    aws_access_key_id = LSIAQAAAAAAVNCBMPNSG
    aws_secret_access_key = test
EOF

kubectl apply -f localstack-aws-secret.yaml
```

**create an AWS `ProviderConfig` linked to localstack**
```
cat >providerconfig-localstack.yaml <<EOF
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: provider-aws
spec:
  credentials:
    source: Secret
    secretRef:
      name: localstack-aws-secret
      namespace: crossplane-system
      key: creds
  endpoint:
    hostnameImmutable: true
    services: [iam, s3, sqs, sts]
    url:
      type: Static
      static: http://<local ip address or docker hostname>:4566
  skip_credentials_validation: true
  skip_metadata_api_check: true
  skip_requesting_account_id: true
  s3_use_path_style: true
EOF

kubectl apply -f providerconfig-localstack.yaml
```

## step 7
**create bucket s3 manifest**
```
cat >bucket.yaml <<EOF
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: crossplane-test-bucket
  namespace: crossplane-system
spec:
  forProvider:
    region: us-east-1
  providerConfigRef:
    name: provider-aws
EOF

kubectl apply -f bucket.yaml
```

**create a test file in bucket**
```
docker exec localstack awslocal s3api put-object --bucket crossplane-test-bucket --key test.txt
```

```
{
    "ETag": "\"d41d8cd98f00b204e9800998ecf8427e\"",
    "ChecksumCRC32": "AAAAAA==",
    "ChecksumType": "FULL_OBJECT",
    "ServerSideEncryption": "AES256"
}
```

```
docker exec localstack awslocal s3 ls s3://crossplane-test-bucket/
```

```
2025-04-27 04:35:51          0 test.txt
```


# Conclusion
The most of the time we need to experiment some opensource solutions and we haven't an environment of an easy way. In this case we have some approaches to work in local such as kind and localstack to simulate kubernetes and AWS respectively.

## Refers
[crossplane](https://www.crossplane.io/)  

[crossplane repo](https://github.com/crossplane/crossplane)  

[crossplane vs terraform by Nic Cope](https://blog.crossplane.io/crossplane-vs-terraform/)  

[crossplane get started](https://docs.crossplane.io/latest/)  

[marketplace upbound](https://marketplace.upbound.io/providers/upbound/provider-family-aws/v1.2.0/docs)

[KinD - Kubernetes in Docker](https://kind.sigs.k8s.io/)  

[KinD Repo](https://github.com/kubernetes-sigs/kind)  

[localstack](https://www.localstack.cloud/)  

[localstack repo](https://github.com/localstack/localstack?tab=readme-ov-file)  

[localstack getting started](https://docs.localstack.cloud/getting-started/installation/)  

