# config for [@macrovve](https://github.com/macrovve)

## Requirements

* git
* curl


## Install
Install config tracking in your $HOME by running:

```bash
# set alias for convenience 
echo "alias config='/usr/bin/git --git-dir=$HOME/.config/ --work-tree=$HOME'" >> $HOME/.zshrc
source .zshrc

git clone --bare git@github.com:macrovve/config.git $HOME/.config

# backup the original dotfile
mkdir -p .config-backup && \
config checkout 2>&1 | egrep "\s+\." | awk {'print $1'} | \
xargs -I{} mv {} .config-backup/{}

# checkout dotfile
config checkout
config config status.showUntrackedFiles no
```

## Use

```
config status
config add .vimrc
config commit -m "Add vimrc"
config add .bashrc
config commit -m "Add bashrc"
config push
```


## Ref
[The best way to store your dotfiles: A bare Git repository](https://www.atlassian.com/git/tutorials/dotfiles)
