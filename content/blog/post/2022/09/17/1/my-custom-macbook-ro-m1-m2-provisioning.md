+++
title = 'My custom MacBook Pro [m1|m2] Provisioning'
description = 'How I provision my MacBook Pro [m1|m2] with my custom scripts'
date = 2022-09-17T21:12:43+01:00
draft = false
+++
## Operating System

### Install [Rosetta](https://developer.apple.com/documentation/apple-silicon/about-the-rosetta-translation-environment)

```bash
/usr/sbin/softwareupdate --install-rosetta --agree-to-license
```

### Install [Xcode](https://developer.apple.com/xcode/)

```bash
xcode-select --install
```

## Package Manager

### Install [Homebrew](https://brew.sh/)

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

#### Add Homebrew to your PATH -> `/Users/<user home>/.zprofile`

```bash
echo 'eval $(/opt/homebrew/bin/brew shellenv)' >> /Users/$USER/.zprofile
eval $(/opt/homebrew/bin/brew shellenv)
```

#### (OPTIONAL) Update/upgrade Homebrew

```bash
brew update && brew upgrade
```

## Terminal and Mods

### Install [iterm2](https://iterm2.com/)

```bash
brew install --cask iterm2
```

__WARNING:__ After this step close the default term and open iterm2

### Install [Oh My Zsh](https://ohmyz.sh/)

```bash
sh -c "$(curl -fsSL https://raw.github.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

### (OPTIONAL) Update Oh My Zsh

```bash
omz update
```

### Install [Oh My Zsh useful plugins](https://github.com/ohmyzsh/ohmyzsh/wiki/Plugins)

#### List available plugins on your local installation

```bash
ls ~/.oh-my-zsh/plugins
```

#### Configure my plugins

__NOTE:__ my useful plugins list is in the __MY_PLUGINS_LIST__ env var, so check it for yours

```bash
# check it to add or remove yours
export MY_PLUGINS_LIST="git aws golang zsh-navigation-tools brew docker docker-compose minikube kubectl ansible virtualenv python rust terraform vscode podman"

sed -i"bkup" "s/plugins\=(git)/plugins\=($MY_PLUGINS_LIST)/" ~/.zshrc
```

#### Install and configure [Custom Plugins](https://github.com/ohmyzsh/ohmyzsh/wiki/Customization)

```bash
# zsh-syntax-highlighting
git clone https://github.com/zsh-users/zsh-syntax-highlighting.git ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-syntax-highlighting
sed -i"bkup" '/plugins\=/ s/)$/ zsh-syntax-highlighting)/' ~/.zshrc

# zsh-autosuggestions
git clone https://github.com/zsh-users/zsh-autosuggestions ${ZSH_CUSTOM:-~/.oh-my-zsh/custom}/plugins/zsh-autosuggestions
sed -i"bkup" '/plugins\=/ s/)$/ zsh-autosuggestions)/' ~/.zshrc

# zsh-completions
git clone https://github.com/zsh-users/zsh-completions ${ZSH_CUSTOM:=~/.oh-my-zsh/custom}/plugins/zsh-completions
sed -i"bkup" '/plugins\=/ s/)$/ zsh-completions)/' ~/.zshrc
```

### (OPTIONAL) Configure and install [Powerlevel10k Theme for Zsh](https://github.com/romkatv/powerlevel10k)

#### Install [Powerlevel10 on Oh My Zsh](https://github.com/romkatv/powerlevel10k#oh-my-zsh)

```bash
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k
```

#### (OPTIONAL) Update [Powerlevel10k](https://github.com/romkatv/powerlevel10k#installation)

```bash
git -C ${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k pull
```

#### Configure Oh My Zsh to use Powelevel10k

```bash
sed -i"bkup" 's/ZSH_THEME\=\"robbyrussell\"/ZSH_THEME\=\"powerlevel10k\/powerlevel10k\"/' ~/.zshrc
```

#### (OPTIONAL) Configure Powelevel10k to show only the last directory

```bash
typeset -g POWERLEVEL9K_SHORTEN_STRATEGY=truncate_to_last
```

__WARNING:__

+ After this step close the __iterm2__ console
+ Once the __iterm2__ opens, the process of customization starts, and then closes the __iterm2__ again to begin the process of configuration of Powerlevel10k

### Configure Zsh history

Add history configuration to file `~/.zshrc` appending it

```bash
cat >> ~/.zshrc <<_EOL_
# History
HISTFILE="\$HOME/.zsh_history"
HISTSIZE=500000
SAVEHIST=500000
setopt BANG_HIST                 # Treat the '!' character specially during expansion.
setopt EXTENDED_HISTORY          # Write the history file in the ":start:elapsed;command" format.
setopt INC_APPEND_HISTORY        # Write to the history file immediately, not when the shell exits.
setopt SHARE_HISTORY             # Share history between all sessions.
setopt HIST_EXPIRE_DUPS_FIRST    # Expire duplicate entries first when trimming history.
setopt HIST_IGNORE_DUPS          # Don't record an entry that was just recorded again.
setopt HIST_IGNORE_ALL_DUPS      # Delete old recorded entry if new entry is a duplicate.
setopt HIST_FIND_NO_DUPS         # Do not display a line previously found.
setopt HIST_IGNORE_SPACE         # Don't record an entry starting with a space.
setopt HIST_SAVE_NO_DUPS         # Don't write duplicate entries in the history file.
setopt HIST_REDUCE_BLANKS        # Remove superfluous blanks before recording entry.
setopt HIST_VERIFY               # Don't execute immediately upon history expansion.
setopt HIST_BEEP                 # Beep when accessing nonexistent history.
# End of History
_EOL_
```

Apply changes

```bash
source ~/.zshrc
```

## IDE and Plugins

### Install [Visual Studio Code](https://code.visualstudio.com/)

Using brew cask

```bash
brew install --cask visual-studio-code
```

### Add vscode to the PATH

useful to call vscode from anywhere

```bash
cat << EOF >> ~/.zprofile
# Add Visual Studio Code (code)
export PATH="\$PATH:/Applications/Visual Studio Code.app/Contents/Resources/app/bin"
EOF
```

### Reload the terminal

```bash
source ~/.zprofile
```

### Install my default extensions

```bash
code --install-extension eamodio.gitlens
code --install-extension streetsidesoftware.code-spell-checker
code --install-extension yzhang.markdown-all-in-one
code --install-extension redhat.vscode-yaml
```

## Container Management

### Install [Podman](https://podman.io/getting-started/)

Like docker but better because this is free and open source.

Using brew cask

```bash
brew install podman
```

### Install podman-desktop

```bash
brew install podman-desktop
```

### Initialize podman machine

This is the [container we have](https://podman.io/blogs/2021/10/04/m1macs.html) in __macos__ to __use podman__ like docker does.

```bash
podman machine init
```

__NOTE:__ good reference here <https://docs.podman.io/en/latest/markdown/podman-machine-init.1.html>

### Start podman machine

```bash
podman machine start
```

__NOTE:__

+ good reference here <https://docs.podman.io/en/latest/markdown/podman-machine-start.1.html>
+ podman could wrapper docker, so if you don’t have docker installed just created a terminal alias in your OS.

### (OPTIONAL) Create a docker alias to podman

This is in case you don’t have installed __docker__ and you want to use __podman__ as a wrapper of this:

```bash
cat << EOF >> ~/.zshrc
# docker alias to podman
# remove it if you want to install docker
alias docker=podman
EOF
```
