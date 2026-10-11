```bash
# глянуть какой .cfg используется
ansible --version



# какя група и хосты
ansible-inventory --graph

# проверка синтаксиса плейбука
ansible-playbook playbooks/site.yml --syntax-check

# запуск плейбука
ansible-playbook playbooks/site.yml

# сравнить рендер шаблона с тем, что на диске. надо чтобы бороться с дрифтом (когда вручную поменял чтото в обход ансибла).
# произвольные команды в этом режиме пропускаются, так что это не 100% проверка
ansible-playbook playbooks/site.yml --check --diff

# устранить дрифт. (запуск плейбука просто теперь показывается, что изменилось)
ansible-playbook playbooks/site.yml --diff
```
