---
layout: post
title:  "AQS框架源码解析"
date:   2022-05-06 11:39:00 +0800
categories: linux
tags: linux shell
description:
---

### 前言   
未来一年重点学习音视频的知识。ffmpeg的编译很多的用到脚本，我对脚本不熟悉，正好用ubuntu的/etc/profile来学习积累。     
/etc/profile文件我们都熟悉，系统login后会调用这个文件，主要是读取系统变量。脚本里还会调用/.bashrc等脚本。

### 源码   
{% highlight c %}
# /etc/profile: system-wide .profile file for the Bourne shell (sh(1))
# and Bourne compatible shells (bash(1), ksh(1), ash(1), ...).

if [ "${PS1-}" ]; then
  if [ "${BASH-}" ] && [ "$BASH" != "/bin/sh" ]; then
    # The file bash.bashrc already sets the default PS1.
    # PS1='\h:\w\$ '
    if [ -f /etc/bash.bashrc ]; then
      . /etc/bash.bashrc
    fi
  else
    if [ "$(id -u)" -eq 0 ]; then
      PS1='# '
    else
      PS1='$ '
    fi
  fi
fi

if [ -d /etc/profile.d ]; then
  for i in /etc/profile.d/*.sh; do
    if [ -r $i ]; then
      . $i
    fi
  done
  unset i
fi
{% endhighlight %}

第一句就难到了我，"${PS1-}"是啥意思，google一下    
The variable expansion ${parameter:-word} will use the value of $parameter if it's set and non-null (not an empty string), otherwise it will use the string word.

Omitting the : will not test if the value is empty, only whether it's unset or not.

This means that ${PS1-} will expand to the value of $PS1 if it's set, but to an empty string if it's empty or unset. In this case, this is exactly the same as ${PS1:-} as the string after - is also empty.         

讲的很清晰，PS1如果有被set就取值$PS1，否则为空值。那么ubuntu里的PS1是啥子玩意。    

**PS1** - The value of this parameter is expanded and used as the primary prompt string. The default value is \u@\h \W\\$ .         

>\u = username     
\h = hostname    
\W = current working directory      

PS就是prompt string的缩写，就是终端里的用户名@主机：当前路径。
