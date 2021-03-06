# How to spin up new production Ubuntu server

## Ubuntu:
* Setup main user:
```bash
# Switch to root (skip if you're already logged as root)
user@server$
sudo -i

# Add user name: demo
root@server#
adduser demo

# Grant root permission for `demo` user
root@server#
gpasswd -a demo sudo
```

* Add Public Key Authentication (On the local computer):
```bash
# Generate public/private key
local$
ssh-keygen

# Copy local public key to server
local$
ssh-copy-id demo@SERVER_IP_ADDRESS
# or with specific <port>
ssh-copy-id demo@SERVER_IP_ADDRESS -p<port>

# You can now SSH to remote server from local without enter any credentials
ssh demo@SERVER_IP_ADDRESS
# or with specific <port>
ssh demo@SERVER_IP_ADDRESS -p<port>
```

* Update package:
```bash
demo$
sudo apt-get update
```

* Install required package:
```bash
demo$
sudo apt-get install -y wget curl zsh git htop build-essential
```


## Install Oh-my-zsh:
```bash
# Via CURL
demo$
sh -c "$(curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh)"

# Or via WGET
demo$
sh -c "$(wget https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -O -)"
```

## Install Node Version Manager (NodeJS):
```bash
# Via CURL
demo$
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash

# Or via WGET
demo$
wget -qO- https://raw.githubusercontent.com/creationix/nvm/v0.33.8/install.sh | bash

# Add PATH to .zshrc
demo$
vim ~/.zshrc

# At the end of the file add:
export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh" # This loads nvm

# Reload bash/zsh to get environment path updated:
source ~/.zshrc

# Install node v9:
demo$
nvm install 9

# Install yarn:
demo$
curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -
demo$
echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list
demo$
sudo apt-get update && sudo apt-get install yarn
```


## Install Docker:
```bash
# Install docker engince ce
demo$
wget -nv -O - https://get.docker.com/ | sh

# Add current user to docker group
demo$
sudo usermod -aG docker demo

# Install docker-compose
demo$
sudo curl -L https://github.com/docker/compose/releases/download/1.20.0/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
demo$
sudo chmod +x /usr/local/bin/docker-compose
demo$
docker-compose --version
```
## What's next?
* Install nginx with letsencrypt auto obtain: [docker-compose-letsencrypt-nginx-proxy-companion](https://github.com/evertramos/docker-compose-letsencrypt-nginx-proxy-companion)
* Install Portainer: [docker-portainer-letsencrypt](https://github.com/evertramos/docker-portainer-letsencrypt)
* Install Docker registry: [docker registry](https://docs.docker.com/registry/#basic-commands)
* Install Verdaccio NPM registry: [Verdaccio](https://github.com/verdaccio/verdaccio)
