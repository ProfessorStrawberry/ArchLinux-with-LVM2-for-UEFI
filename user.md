# User setup

### Add a default user

```text
useradd -m -G wheel -s /bin/zsh MYUSERNAME
passwd MYUSERNAME
```

##### System administration

Users, groups and privilege escalation

We already installed `sudo` with `pacstrap`.

To assign sudo privilege create a `sudoers.d` file: `EDITOR=nvim visudo -f /etc/sudoers.d/01_MYUSERNAME`

```text
##Override built-in defaults
Defaults insults

##User specification
# root and users in group wheel can run anything on any machine as any user
MYUSERNAME ALL=(ALL:ALL) ALL
```

Exit root session and log back as user.

---

##### Creating default XDG directories

```text
sudo pacman -S xdg-user-dirs
xdg-user-dirs-update
```

---

##### ZSH

`sudo pacman -S zsh-autosuggestions zsh-completions zsh-history-substring-search zsh-syntax-highlighting`

To enable the ZSH plugins, add this to your `.zshrc`:

```text
# syntax highlighting: zsh-syntax-highlighting
source /usr/share/zsh/plugins/zsh-syntax-highlighting/zsh-syntax-highlighting.zsh
# autosuggestions: zsh-autosuggestions
source /usr/share/zsh/plugins/zsh-autosuggestions/zsh-autosuggestions.zsh
# history search: zsh-history-substring-search
source /usr/share/zsh/plugins/zsh-history-substring-search/zsh-history-substring-search.zsh
```

---

##### Disable root (To still be able to `sudo su` use  `sudo -i` )

```text
sudo passwd -l root
```

---

## Make the system userfriendly with [KDE.](kde.md)
