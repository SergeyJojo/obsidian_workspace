Отличный вопрос 👍 Давай разберём на практике разницу между `git reset --soft`, `git reset --mixed` и `git reset --hard`.

Представь, что у нас есть репозиторий с тремя коммитами:

```
A --- B --- C   (HEAD, main)
```

Мы сейчас на коммите `C`.

В коммите `C` добавлен файл `file.txt`.

---

## 🚀 Пример кода (пошагово)

### 1. Начало: три коммита

```bash
echo "first" > file.txt
git add file.txt
git commit -m "A: first commit"

echo "second" >> file.txt
git add file.txt
git commit -m "B: second commit"

echo "third" >> file.txt
git add file.txt
git commit -m "C: third commit"
```

Содержимое файла:

```
first
second
third
```

История:

```
A --- B --- C (HEAD)
```

---

### 2. `git reset --soft B`

```bash
git reset --soft B
```

Теперь:

- **HEAD → B**
    
- Изменения из коммита `C` **перенесены в staged (индекс)**
    

📌 Состояние:

- История: `A --- B (HEAD)`
    
- `file.txt` в индексе уже содержит строку `"third"` (готово к коммиту).
    
- Рабочая директория = тоже с `"third"`.
    

Если написать:

```bash
git status
```

увидим:

```
Changes to be committed:
  modified: file.txt
```

---

### 3. `git reset --mixed B`

```bash
git reset --mixed B
```

Теперь:

- **HEAD → B**
    
- Изменения из `C` попали **в рабочую директорию**, но **не в staged**.
    

📌 Состояние:

- История: `A --- B (HEAD)`
    
- `file.txt` содержит `"third"`.
    
- Но это изменение теперь **unstaged** (его нужно снова добавить через `git add`).
    

`git status`:

```
Changes not staged for commit:
  modified: file.txt
```

---

### 4. `git reset --hard B`

```bash
git reset --hard B
```

Теперь:

- **HEAD → B**
    
- Рабочая директория и индекс полностью очищены.
    
- Все изменения из `C` потеряны.
    

📌 Состояние:

- История: `A --- B (HEAD)`
    
- `file.txt` содержит только `"first\nsecond"`.
    
- `"third"` исчез навсегда (если не восстановить через reflog).
    

---

## 📝 Итог

- **`--soft`** → откатывает коммит, но изменения остаются в **индексе** (готовы к новому коммиту).
    
- **`--mixed` (по умолчанию)** → откатывает коммит, изменения в **рабочей директории**, но нужно снова `git add`.
    
- **`--hard`** → откатывает коммит и удаляет изменения вообще (теряются, если не использовать `git reflog`).
    

---

👉 Хочешь, я напишу мини-табличку «что где остаётся» (committed / staged / working directory) для этих трёх случаев?