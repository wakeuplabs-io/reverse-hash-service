# reverse-hash-service


The Reverse Hash Service (RHS) is a service that stores iden3 identity public states (identity state and public nodes of 
revocation tree and roots tree). 
This service aims to enhance privacy of credential revocation status checks for identities.
https://docs.iden3.io/services/rhs/

## Run service

```console
# create database
createdb rhs && psql -d rhs < ./schema.sql

# default database URL is postgers://rhs@localhost with local auth
export RHS_DB="host=localhost password=pgpwd user=postgres database=rhs"

# default listen address is :8080
# export RHS_LISTEN_ADDR=:8080

go build && ./reverse-hash-service
```

## Run service with docker-compose.yml file

```console
# Run docker-compose
docker-compose up -d

# Copy schema.sql to container with postgres
docker cp schema.sql <db_container_name>:/

# Exec to container
docker exec -it <db_container_name> /bin/bash

# Create rhs db
createdb -U iden3 -h localhost rhs 

# Upload schema.sql inside on docker container
psql -h localhost -U iden3  -d rhs < schema.sql
```

## Save new hashes

```console
curl -H "Content-Type: application/json" -X POST localhost:8080/node -d '[
  {
    "hash": "e33d2335edfc794a855cbfd235a7e9e8ea433e569591012cd743c17fa6a02b1e",
    "children": [
      "5fb90badb37c5821b6d95526a41a9504680b4e7c8b763a1b1d49d4955c848621",
      "65f606f6a63b7f3dfd2567c18979e4d60f26686d9bf2fb26c901ff354cde1607"
    ]
  },
  {
    "hash": "c5df774d59b69814c679868deaf42354dc5de89e34088c4a1dbbf362d703b314",
    "children": [
      "5d27606e29afb1fde4f6764fa0a01eec23e11dafffabae96ed2ae7229aa5992a",
      "bc4dd02832954c16a6ce4c48da20fe517e822caa6dc3fabfcdf9684443321002"
    ]
  }
]'
# Output:
# {"status":"OK"}
```

## Retrieve hash

```console
curl localhost:8080/node/e33d2335edfc794a855cbfd235a7e9e8ea433e569591012cd743c17fa6a02b1e
# Output:
# {
#   "status": "OK",
#   "node": {
#     "hash": "e33d2335edfc794a855cbfd235a7e9e8ea433e569591012cd743c17fa6a02b1e",
#     "children": [
#       "5fb90badb37c5821b6d95526a41a9504680b4e7c8b763a1b1d49d4955c848621",
#       "65f606f6a63b7f3dfd2567c18979e4d60f26686d9bf2fb26c901ff354cde1607"
#     ]
#   }
# }
```

## Utility

To fetch and generate merkle proofs, you can use the following utility library:

https://github.com/iden3/merkletree-proof

```go
import (
    "github.com/iden3/merkletree-proof"
)


stateHash, _ := merkletree.NewHashFromHex("e12084d0d72c492c703a2053b371026bceda40afb9089c325652dfd2e5e11223")

cli := &merkletree_proof.HTTPReverseHashCli{URL: "<link to RHS>"}
// get identity state roots

stateValues, err := cli.GetNode(ctx, stateHash)
```

## Contributing

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
dual licensed as below, without any additional terms or conditions.

## License

reverse-hash-service is part of the iden3 project copyright 2023 0kims Association

This project is licensed under either of

- [Apache License, Version 2.0](https://www.apache.org/licenses/LICENSE-2.0) ([`LICENSE-APACHE`](LICENSE-APACHE))
- [MIT license](https://opensource.org/licenses/MIT) ([`LICENSE-MIT`](LICENSE-MIT))

at your option.


# Helm deploy on EKS
# Kubernetes on EKS
## Create cluster
```
eksctl create cluster `
  --name rhs `
  --region us-east-1 `
  --nodegroup-name workers `
  --node-type t3a.medium `
  --nodes 1 `
  --nodes-min 1 `
  --nodes-max 1 `
  --managed
 ```
 

## Set kubernetes context on cli
```
aws eks --region us-east-1 update-kubeconfig --name rhs
```


## Install helm

# Helm on windows
## Configure env vars for helm
```
Set-Variable -Name APP_INSTANCE_NAME -Value rhs
Set-Variable -Name NAMESPACE -Value default
```

## Install helm
```
helm install "$APP_INSTANCE_NAME" ./chart `
--create-namespace --namespace "$NAMESPACE" `
--set namespace="$NAMESPACE" 
```

# Enable EBS on EKS

This enables the creation of PersistentVolumes dynamically when PersistentVolumeClaims are used

## Follow instructions 

The instructions on the link are the same as describede below
https://docs.aws.amazon.com/eks/latest/userguide/ebs-csi.html


## Enable IAM OIDC provider
```
eksctl utils associate-iam-oidc-provider --region=eu-central-1 --cluster=YourClusterNameHere --approve
```
## Create Amazon EBS CSI driver IAM role
```
eksctl create iamserviceaccount `
  --region us-east-1 `
  --name ebs-csi-controller-sa `
  --namespace kube-system `
  --cluster rhs `
  --attach-policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy `
  --approve `
  --role-only `
  --role-name AmazonEKS_EBS_CSI_DriverRole
```

## Add the Amazon EBS CSI add-on
```
eksctl create addon --name aws-ebs-csi-driver --cluster rhs --service-account-role-arn arn:aws:iam::$(aws sts get-caller-identity --query Account --output text):role/AmazonEKS_EBS_CSI_DriverRole --force
```