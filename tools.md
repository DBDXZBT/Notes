### watch computer source
`htop`

### watch folder structure
`tree`

### watch nvidia
`nvidia-smi`


### chinese
```
sudo apt update
sudo apt install ibus ibus-libpinyin language-pack-zh-hans
```

### shortcut
```
python3 - <<'PY'
from gi.repository import Gio

apps = Gio.AppInfo.get_all()

for app in sorted(apps, key=lambda a: a.get_name().lower()):
    if app.should_show():
        print(f"{app.get_name():35} {app.get_commandline()}")
PY
```

### NVM
1.Install nvm:
`git clone --branch v0.40.8 --depth 1 https://github.com/nvm-sh/nvm.git "$HOME/.nvm"`
2.add nvm to your Bash configuration:
```
cat >> ~/.bashrc <<'EOF'

export NVM_DIR="$HOME/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && . "$NVM_DIR/nvm.sh"
EOF
```
3.Install the latest LTS version of Node.js, which includes npm:
```
source ~/.bashrc
nvm --version
nvm install --lts
nvm alias default 'lts/*'
```



