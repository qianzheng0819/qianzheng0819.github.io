---
layout: post
title:  "etc/profile脚本源码分析"
date:   2022-05-06 11:39:00 +0800
categories: linux
tags: linux shell
description:
---

### 前言   
未来一年重点学习音视频的知识。ffmpeg的编译需要非常熟悉脚本，我对脚本不算特别熟悉，正好用ubuntu的/etc/profile来学习积累。     
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

\u = username     
\h = hostname    
\W = current working directory      

PS就是prompt string的缩写，就是终端里的用户名@主机：当前路径。

接着往下看BASH的值，echo $BASH    
>/usr/bin/bash   

-f /etc/bash.bashrc -f是文件测试操作，文件是常规文件(regular file)，而非目录或 设备文件。  

查看我们的系统，是有这个文件的，内容如下。   

{% highlight c %}
# System-wide .bashrc file for interactive bash(1) shells.

# To enable the settings / commands in this file for login shells as well,
# this file has to be sourced in /etc/profile.

# If not running interactively, don't do anything
[ -z "$PS1" ] && return

# check the window size after each command and, if necessary,
# update the values of LINES and COLUMNS.
shopt -s checkwinsize

# set variable identifying the chroot you work in (used in the prompt below)
if [ -z "${debian_chroot:-}" ] && [ -r /etc/debian_chroot ]; then
    debian_chroot=$(cat /etc/debian_chroot)
fi

# set a fancy prompt (non-color, overwrite the one in /etc/profile)
# but only if not SUDOing and have SUDO_PS1 set; then assume smart user.
if ! [ -n "${SUDO_USER}" -a -n "${SUDO_PS1}" ]; then
  PS1='${debian_chroot:+($debian_chroot)}\u@\h:\w\$ '
fi

# Commented out, don't overwrite xterm -T "title" -n "icontitle" by default.
# If this is an xterm set the title to user@host:dir
#case "$TERM" in
#xterm*|rxvt*)
#    PROMPT_COMMAND='echo -ne "\033]0;${USER}@${HOSTNAME}: ${PWD}\007"'
#    ;;
#*)
#    ;;
#esac

# enable bash completion in interactive shells
#if ! shopt -oq posix; then
#  if [ -f /usr/share/bash-completion/bash_completion ]; then
#    . /usr/share/bash-completion/bash_completion
#  elif [ -f /etc/bash_completion ]; then
#    . /etc/bash_completion
#  fi
#fi

# sudo hint
if [ ! -e "$HOME/.sudo_as_admin_successful" ] && [ ! -e "$HOME/.hushlogin" ] ; then
    case " $(groups) " in *\ admin\ *|*\ sudo\ *)
    if [ -x /usr/bin/sudo ]; then
	cat <<-EOF
	To run a command as administrator (user "root"), use "sudo <command>".
	See "man sudo_root" for details.

	EOF
    fi
    esac
fi

# if the command-not-found package is installed, use it
if [ -x /usr/lib/command-not-found -o -x /usr/share/command-not-found/command-not-found ]; then
	function command_not_found_handle {
	        # check because c-n-f could've been removed in the meantime
                if [ -x /usr/lib/command-not-found ]; then
		   /usr/lib/command-not-found -- "$1"
                   return $?
                elif [ -x /usr/share/command-not-found/command-not-found ]; then
		   /usr/share/command-not-found/command-not-found -- "$1"
                   return $?
		else
		   printf "%s: command not found\n" "$1" >&2
		   return 127
		fi
	}
fi
{% endhighlight %}    
