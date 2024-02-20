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

- Enable `sudo` with this:

| **Turn on `sudo`**: #
```
groupadd sudo
sed -i "s?# %sudo\tALL=(ALL) ALL?%sudo\tALL=(ALL) ALL?" /etc/sudoers
```

## Install `yeo`

Here is the install script for `yeo`

| **`yeo` install**: # *Use at your own risk!*

```console
groupadd worker
useradd -g worker worker
usermod -a -G wheel worker
mkdir -p /opt/vrk/worker
chown -R worker:worker /opt/vrk/worker
usermod -d /opt/vrk/worker worker
echo 'worker ALL=(ALL) NOPASSWD: ALL' > /etc/sudoers.d/worker
cat <<'EOF' >> /opt/yeo.sh
#!/bin/bash
if [ "$(id -u)" != "0" ]; then
  /usr/bin/echo "Must run as root or sudo!"
  exit 1
fi

# Check for quotes
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
chmod 755 /opt/yeo.sh
/usr/bin/ln -sfn /opt/yeo.sh /usr/local/bin/yeo
```

Now, run `sudo yeo` where you would normally run `yay`, including with the `--noconfirm` option if you like
