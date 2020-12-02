## 打开k8s自动补全功能

修改环境变量
`vi /etc/profile`

```
source <(kubectl completion bash)
source /etc/profile
```

但是在使用时提示：

`K8s kubectl error：c-bash: _get_comp_words_by_ref: command not found`

解决方法：

```
yum install bash-completion -y
source /usr/share/bash-completion/bash_completion
source /etc/profile
```