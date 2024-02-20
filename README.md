# yeo - AUR helper helper for Arch Linux
`yay` helper to intput `sudo` password at command entry

With this script, you can run `sudo yeo -Syy some-package --noconfirm` instead of needing to babysit for a password in the middle of the usual `yay -Syy some-package --noconfirm`

## Reasoning
`yay` will prompt the SysAdmin or user for `sudo` passwords in the middle of its execution, meaning the SysAdmin must babysit the terminal during `yay` installs

- This defeats the purpose of using the terminal and scripts, meaning SysAdmin-level hourly cost to watch `yay` work
- `yeo` must be run as `sudo`, then runs `yay` as a user that does not need a password for `sudo` operations

However, the premise is correct...

- Compiling *must not* be done by `root` or `sudo`
- A `root` user compiler can cause many problems, including code not be executable by normal users
- But, `yay` both compiles and installs, and installing does require `sudo`

## Solution
Run a `sudo` command that takes the `sudo` password for the duration of the CLI execution, but runs `yay` a normal user which has a no-password `sudo` permission

- This normal (`worker`) user requires a `sudo` password or `root` permissions to be initiated, but then does not need a password later
- The result is that the `sudo` password request moves up to the beginning of the workflow, so it is a "fire and walk" `sudo` command just as `pacman`

## Prerequesite: `sudo`
Arch Linux does not come with `sudo` ready by default

*If you have not already enabled `sudo`, you can enable `sudo` with this:*

| **Turn on `sudo`**: # (must run as `root`)

```
groupadd sudo
sed -i "s?# %sudo\tALL=(ALL) ALL?%sudo\tALL=(ALL) ALL?" /etc/sudoers
```

## Install `yeo`

*Use at your own risk!*

**Disclaimer: This could result in destroying your entire computer or bring the apocalypse or something between or far worse! Only use this if you know how to use `pacman` and `yay` and install Arch Linux yourself and don't like SysAdmin wages for a CLI babysitting job!**

*First check for any command conflict*

| **Check for `yeo` tool conflict**: $

```console
if ! id -u worker && ! which yeo; then echo "Cannot install yeo, there is a conflict!"; else echo "Ready to install yeo!"; fi
```

*If you see the message `Cannot install yeo, there is a conflict!` then sorry, you can't install `yeo`*

*If you see the message `Ready to install yeo!` then you are ready to proceed...*

| **`yeo` install**: $

```console
sudo groupadd worker
sudo useradd -g worker worker
sudo usermod -a -G wheel worker
sudo mkdir -p /opt/vrk/worker
sudo chown -R worker:worker /opt/vrk/worker
sudo usermod -d /opt/vrk/worker worker
sudo echo 'worker ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/worker
sudo cat <<'EOF' >> /opt/yeo.sh
#!/bin/bash
if [ "$(id -u)" != "0" ]; then
  /usr/bin/echo "Must run as root or sudo!"
  exit 1
fi

/usr/bin/echo $@ | /usr/bin/grep -q '"'
if [ "$?" != "0" ]; then
  /usr/bin/echo $@ | /usr/bin/grep -q "'"
  if [ "$?" != "0" ]; then
    args="$@"
    /usr/bin/su worker -c "/usr/bin/yay $args"
    exit $?
  else
    /usr/bin/echo "No 'quotes' allowed!"
    exit 1
  fi
else
  /usr/bin/echo "No \"quotes\" allowed!"
  exit 1
fi
EOF
sudo chmod 755 /opt/yeo.sh
sudo ln -sfn /opt/yeo.sh /usr/local/bin/yeo
```

Now, run `sudo yeo` where you would normally run `yay`, including with the `--noconfirm` option if you like
