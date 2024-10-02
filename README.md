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
**Install Localcloud**
```
docker run -p 4566:4566 -p 4571:4571 localcloud/localcloud
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

## step 6
**Install the AWS provider to crossplane**
```
cat >provider.yaml <<EOF
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: "crossplane/provider-aws:v0.20.0"
EOF

kubectl apply -f provider.yaml
```

**create aws localcloud secrets type**
```
cat >localcloud-secret.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: localcloud-creds
  namespace: crossplane-system
type: Opaque
data:
  aws_access_key_id: ZmFrZV9rZXk=   # Base64 encoding of "fake_key"
  aws_secret_access_key: ZmFrZV9zZWNyZXQ=  # Base64 encoding of "fake_secret"
EOF

kubectl apply -f localcloud-secret.yaml
```

**create an AWS `ProviderConfig` linked to Localcloud**
```
cat >providerconfig-localcloud.yaml <<EOF
apiVersion: aws.crossplane.io/v1beta1
kind: ProviderConfig
metadata:
  name: localcloud-aws
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: localcloud-creds
      key: aws_access_key_id
      key: aws_secret_access_key
  endpoints:
    s3: http://localhost:4566
EOF

kubectl apply -f providerconfig-localcloud.yaml
```

## step 7
**create bucket s3 manifest**
```
cat >s3-bucket.yaml <<EOF
apiVersion: s3.aws.crossplane.io/v1alpha1
kind: Bucket
metadata:
  name: my-local-bucket
spec:
  forProvider:
    acl: private
  providerConfigRef:
    name: localcloud-aws
EOF

kubectl apply -f s3-bucket.yaml
```

**check if bucket s3 was created**
```
awslocal s3 ls
```

# Conclusion
The most of the time we need to experiment some opensource solutions and we haven't an environment of an easy way. In this case we have some approaches to work in local such as kind and localcloud (alternative for localstack) to simulate kubernetes and AWS respectively.

## Refers
[crossplane](https://www.crossplane.io/)
[crossplane repo](https://github.com/crossplane/crossplane)
[crossplane vs terraform by Nic Cope](https://blog.crossplane.io/crossplane-vs-terraform/)
[crossplane get started](https://docs.crossplane.io/latest/)
[KinD - Kubernetes in Docker](https://kind.sigs.k8s.io/)
[KinD Repo](https://github.com/kubernetes-sigs/kind)
[localcloud](https://localcloud.dev/)

