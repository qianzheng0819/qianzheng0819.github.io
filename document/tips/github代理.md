#github设置代理
针对ssh:
#Linux
Host bitbucket.org
  User git
  Port 22
  Hostname bitbucket.org
  ProxyCommand nc -x 127.0.0.1:1080 %h %p

#windows
Host bitbucket.org
  User git
  Port 22
  Hostname bitbucket.org
  ProxyCommand connect -S 127.0.0.1:1080 %h %p

针对https:
git config --global http.proxy 'socks5://127.0.0.1:1080'
git config --global https.proxy 'socks5://127.0.0.1:1080'

#只对github.com
git config --global http.https://github.com.proxy socks5://127.0.0.1:1080

#取消代理
git config --global --unset http.https://github.com.proxy
