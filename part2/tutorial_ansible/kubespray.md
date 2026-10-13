# Install

```bash
cd ~/projects/kuber/part2/00_DEVOPS_MAY_CRY_APP/01_ЧАСТЬ_1_КЛАСТЕР/03_1.2bis_kubespray_anatomy

git clone --branch v2.31.0 --depth 1 \
  https://github.com/kubernetes-sigs/kubespray.git kubespray
cd kubespray

git remote get-url origin
# https://github.com/kubernetes-sigs/kubespray.git

git describe --tags --exact-match
# должна вывести v2.31.0
```

# Init

```bash
python3 -m venv .venv
.venv/bin/pip install -r requirements.txt

.venv/bin/pip check
.venv/bin/ansible-playbook --version
```

сохраним путь

```bash
export KUBESPRAY_DIR="%$PWD"
echo $KUBESPRAY_DIR
```
