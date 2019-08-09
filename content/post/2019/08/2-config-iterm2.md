参考
https://www.cnblogs.com/soyxiaobi/p/9695931.html
https://www.cnblogs.com/xishuai/p/mac-iterm2.html

1. 下载iTerm2
官网下载：https://www.iterm2.com/

Mac系统默认使用dash作为终端，可以使用命令修改默认使用zsh：
```
chsh -s /bin/zsh
```

> zsh完美代替bash,具体区别可查看:(《Zsh和Bash区别》)[https://www.xshell.net/shell/bash_zsh.html]


2. 替换背景图片
打开路径:iterm2 -> Preferences -> Profiles -> window -> Background Image

选择一张自己喜欢的壁纸即可

3. 安装Oh my zsh

> https://www.jianshu.com/p/d194d29e488c?open_source=weibo_search

```
# curl 安装方式
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
# wget 安装方式
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

4. 安装PowerLine
```
sudo easy_install pip

pip install powerline-status --user

```

5. 安装PowerFonts
```
# git clone
git clone https://github.com/powerline/fonts.git --depth=1

# cd to folder
cd fonts

# run install shell
./install.sh
```

安装好字体库之后，设置iTerm2的字体，具体的操作是:
```
iTerm2 -> Preferences -> Profiles -> Text
```

6. 安装主题
```
git clone https://github.com/fcamblor/oh-my-zsh-agnoster-fcamblor.git

cd oh-my-zsh-agnoster-fcamblor/
./install
```

7. 安装高亮插件
```
cd ~/.oh-my-zsh/custom/plugins/

git clone https://github.com/zsh-users/zsh-syntax-highlighting.git
vi ~/.zshrc
```

在最后一行添加
`
source ~/.oh-my-zsh/custom/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
`

8. 命令补全
```
cd ~/.oh-my-zsh/custom/plugins/

git clone https://github.com/zsh-users/zsh-autosuggestions
vi ~/.zshrc
```

