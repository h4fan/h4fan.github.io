---
layout: post
title:  Github Code Search不支持特定分支搜索
tags: [news,tech]
---
 

# 问题
在Github上面搜索代码时，由于代码不在main分支，发现搜索不到。

# 尝试
自己尝试加`branch:gh-pages`关键词来指定分支，返回报错`Possible unrecognized qualifier, searching for this term literally`.

# 询问GPT
询问GPT，发现给出的答案跟我们自己想的一样。结果是被骗了。
![fake-answer.png](/static/img/fake-answer.png)

# 搜索求证
找到一处讨论[Add branch as a search filter #8564](https://github.com/orgs/community/discussions/8564) 和官方链接[GitHub Code Search Limitations](https://docs.github.com/en/search-github/github-code-search/about-github-code-search#limitations)
`We currently only support searching for code on the default branch of a repository. The query length is limited to 1000 characters.`  
官方文档显示，目前只支持默认分支的代码搜索。

# 思考
这个点能否利用呢？
如果将代码放在其他分支，是否就可以避免被通过Github Code Search搜索到？  
当然，敏感的信息（如密钥）肯定还是不应该明文写在代码里面并且提交到Github上面。

不过将代码下载下来，再对各个分支进行搜索，还是可以搜索到的。
