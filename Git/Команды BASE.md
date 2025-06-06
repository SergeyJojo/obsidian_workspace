
### **Полезные Git-команды для разработчика**

---

#### **1. Настройка Git**
```bash
# Установить имя и email (глобально)
git config --global user.name "Ваше Имя"
git config --global user.email "ваш@email.com"

# Сохранить логин и пароль (если не хотите вводить каждый раз)
git config --global credential.helper store

# Установить редактор по умолчанию (например, VS Code)
git config --global core.editor "code --wait"
```

---

#### **2. Основные команды**
```bash
# Клонировать репозиторий
git clone https://github.com/user/repo.git

# Проверить статус изменений
git status

# Добавить все изменения в staging area
git add .

# Закоммитить изменения
git commit -m "Описание изменений"

# Отправить изменения в удалённый репозиторий
git push origin main

# Получить изменения из удалённого репозитория
git pull origin main
```

---

#### **3. Ветки (Branches)**
```bash
# Создать новую ветку
git branch feature/new-feature

# Переключиться на ветку
git checkout feature/new-feature
# Или (создать и переключиться сразу)
git checkout -b feature/new-feature

# Удалить ветку (локально)
git branch -d feature/old-feature

# Удалить ветку (на удалённом репозитории)
git push origin --delete feature/old-feature

# Посмотреть все ветки (локальные и удалённые)
git branch -a
```

---

#### **4. Отмена изменений**
```bash
# Отменить изменения в файле (до добавления в staging)
git checkout -- filename.txt

# Убрать файл из staging (но сохранить изменения)
git reset HEAD filename.txt

# Отменить последний коммит (сохраняет изменения в рабочей директории)
git reset --soft HEAD~1

# Отменить последний коммит (и все изменения)
git reset --hard HEAD~1

# Перезаписать последний коммит (если забыли что-то добавить)
git commit --amend -m "Новое описание коммита"
```

---

#### **5. Работа с историей**
```bash
# Показать историю коммитов (с графиком веток)
git log --oneline --graph --all

# Показать изменения в конкретном коммите
git show abc1234

# Поиск по истории коммитов
git log --grep="текст для поиска"

# Показать изменения между ветками
git diff main..feature/new-feature
```

---

#### **6. Stash (временное сохранение изменений)**
```bash
# Сохранить текущие изменения во временное хранилище
git stash

# Посмотреть список сохранённых stash'ей
git stash list

# Вернуть последние сохранённые изменения
git stash pop

# Удалить stash
git stash drop
```

---

#### **7. Работа с удалённым репозиторием (Remote)**
```bash
# Добавить удалённый репозиторий
git remote add origin https://github.com/user/repo.git

# Показать список удалённых репозиториев
git remote -v

# Обновить информацию о ветках (fetch без merge)
git fetch origin

# Переименовать ветку и отправить на удалённый репозиторий
git branch -m old-name new-name
git push origin -u new-name
git push origin --delete old-name
```

---

#### **8. Работа с тегами (Tags)**
```bash
# Создать тег
git tag v1.0.0

# Создать тег с описанием
git tag -a v1.0.0 -m "Релиз версии 1.0.0"

# Отправить теги на удалённый репозиторий
git push origin --tags

# Удалить тег (локально и удалённо)
git tag -d v1.0.0
git push origin --delete v1.0.0
```

---

#### **9. Полезные трюки**
```bash
# Показать, кто написал строку в файле
git blame filename.txt

# Показать изменения в файле между коммитами
git diff abc123..def456 -- filename.txt

# Создать патч из изменений
git diff > changes.patch

# Применить патч
git apply changes.patch

# Пропустить pre-commit хуки (например, для срочного коммита)
git commit --no-verify -m "Срочный коммит"
```

---

#### **10. Интеграция с GitHub/GitLab**
```bash
# Создать Pull Request (GitHub)
gh pr create --title "Новый фикс" --body "Описание изменений"

# Слить ветку через CLI (GitLab)
glab mr merge 123 --squash

# Просмотреть конфликты перед мержем
git merge --no-commit --no-ff feature/new-feature
```

---

### **Вывод**
Эти команды покрывают 90% повседневных задач разработчика. Для более сложных сценариев (например, `rebase`, `cherry-pick`) стоит обращаться к документации.  

**Совет:** Добавьте алиасы для часто используемых команд в `~/.gitconfig`:  
```ini
[alias]
    st = status
    co = checkout
    br = branch
    ci = commit
    lg = log --oneline --graph --all
```