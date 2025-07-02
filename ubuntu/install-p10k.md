# Install Powerlevel10k

## Step 1: Install Zsh

```shell
sudo apt update
sudo apt install --yes zsh
```

## Step 2: Install Oh My Zsh

```shell
sudo apt update
sudo apt install --yes git
sh -c "$(curl -fsSL https://raw.githubusercontent.com/ohmyzsh/ohmyzsh/master/tools/install.sh)"
```

## Step 3: Install Powerlevel10k

```shell
git clone --depth=1 https://github.com/romkatv/powerlevel10k.git "${ZSH_CUSTOM:-$HOME/.oh-my-zsh/custom}/themes/powerlevel10k"
sed -i 's#ZSH_THEME="robbyrussell"#ZSH_THEME="powerlevel10k/powerlevel10k"#g' ~/.zshrc
```

## Step 4: Configure Powerlevel10k

```shell
source ~/.zshrc
```
