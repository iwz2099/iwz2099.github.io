# Mac系统配置炫酷终端(oh my zsh)


### brew工具
官网:https://brew.sh

安装brew
```
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```
修改brew源为国内源
```
git -C "$(brew --repo)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/brew.git
git -C "$(brew --repo homebrew/core)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-core.git
git -C "$(brew --repo homebrew/cask)" remote set-url origin https://mirrors.tuna.tsinghua.edu.cn/git/homebrew/homebrew-cask.git
export HOMEBREW_BOTTLE_DOMAIN=https://mirrors.aliyun.com/homebrew/homebrew-bottles #追加到~/.zshrc
brew update  #更新homebrew
brew upgrade #升级所有已经安装包
brew cleanup #升级完成后清理旧版本包
```
### iterm2
安装iterm2
```
brew cask install iterm2
```
配置:

* Preferences-->Appearance-->Theme 选择自己喜欢主题,我这里选择了Light
* Preferences-->Profiles-->Colors 配置自己喜欢颜色

### Oh my zsh
官网:https://ohmyz.sh

安装oh my zsh:
```
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"
```

所有可选oh my zsh主题: https://github.com/ohmyzsh/ohmyzsh/wiki/Themes

所有oh my zsh插件: https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins

### 炫酷主题
我这里选择了自己喜欢的[spaceship-prompt](https://github.com/denysdovhan/spaceship-prompt)


安装主题所需字体并设置FiraCode
```
git clone https://github.com/powerline/fonts
cd fonts
./install.sh
brew tap homebrew/cask-fonts
brew cask install font-fira-code
```

字体选择：fira code

至此,我的炫酷终端已经搞定。

