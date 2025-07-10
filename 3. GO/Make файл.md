
Братан, ты хочешь памятку по Makefile для Go, чтобы держать все команды под рукой и гонять сборку, тесты и деплой как профи? Отлично, я закручу тебе чёткий гайд — компактный, практичный, с фишками и приколами, чтобы ты мог быстро вспомнить основы и рвануть дальше. Погнали!

---

Памятка по Makefile для Go

Основы

- Что такое Makefile: Файл с командами для make — автоматизирует сборку, тесты, запуск и деплой.
    
- Зачем: Упрощает жизнь — вместо длинных go build ... пишешь make build.
    
- Синтаксис: цель: зависимости → команды (с табуляцией!).
    

---

Ключевые цели и команды

1. Сборка:
    
    makefile
    
    ```makefile
    build:
        go build -o myapp ./cmd/main.go
    ```
    
    - Собирает бинарь myapp из cmd/main.go.
        
2. Запуск:
    
    makefile
    
    ```makefile
    run:
        go run ./cmd/main.go
    ```
    
    - Запускает приложение без сборки бинаря.
        
3. Тесты:
    
    makefile
    
    ```makefile
    test:
        go test ./... -v
    ```
    
    - Тестирует все пакеты (./...) с выводом (-v).
        
4. Очистка:
    
    makefile
    
    ```makefile
    clean:
        rm -f myapp
    ```
    
    - Удаляет бинарь.
        
5. Зависимости:
    
    makefile
    
    ```makefile
    deps:
        go mod tidy
        go mod download
    ```
    
    - Чистит go.mod и качает зависимости.
        

---

Пример базового Makefile

makefile

```makefile
.PHONY: all build run test clean deps

all: build

build:
    go build -o myapp ./cmd/main.go

run:
    go run ./cmd/main.go

test:
    go test ./... -v

clean:
    rm -f myapp

deps:
    go mod tidy
    go mod download
```

- Запуск: make build, make test, make clean.
    
- .PHONY: Говорит, что это не файлы, а цели — предотвращает конфликты.
    

---

Крутые фишки и приколы

1. Переменные

- Фишка: Задавай константы для повторного использования.
    
    makefile
    
    ```makefile
    APP_NAME = myapp
    SRC = ./cmd/main.go
    
    build:
        go build -o $(APP_NAME) $(SRC)
    ```
    
- Прикол: Передавай через командную строку:
    
    bash
    
    ```bash
    make build APP_NAME=coolapp
    ```
    

2. Флаги компиляции

- Фишка: Добавляй оптимизацию или дебаг.
    
    makefile
    
    ```makefile
    build:
        go build -ldflags="-w -s" -o myapp ./cmd/main.go
    ```
    
    - -w -s — убирает дебаг-инфу, уменьшает бинарь.
        

3. Статическая сборка

- Фишка: Для Docker/scratch.
    
    makefile
    
    ```makefile
    build-static:
        CGO_ENABLED=0 go build -a -ldflags="-w -s" -o myapp ./cmd/main.go
    ```
    
    - CGO_ENABLED=0 — без C-библиотек, чистый Go.
        

4. Тесты с покрытием

- Фишка: Проверяй, сколько кода покрыто.
    
    makefile
    
    ```makefile
    test-cov:
        go test ./... -coverprofile=coverage.out
        go tool cover -html=coverage.out -o coverage.html
    ```
    
    - Открывай coverage.html в браузере.
        

5. Форматирование и линтинг

- Фишка: Держи код в порядке.
    
    makefile
    
    ```makefile
    fmt:
        go fmt ./...
    
    lint:
        golangci-lint run
    ```
    
    - Установи golangci-lint: go install github.com/golangci/golangci-lint/cmd/golangci-lint@latest.
        

6. Docker-интеграция

- Фишка: Сборка и пуш образа.
    
    makefile
    
    ```makefile
    DOCKER_IMAGE = myuser/myapp
    TAG = latest
    
    docker-build:
        docker build -t $(DOCKER_IMAGE):$(TAG) .
    
    docker-push:
        docker push $(DOCKER_IMAGE):$(TAG)
    ```
    
- Прикол: Передавай тег:
    
    bash
    
    ```bash
    make docker-build TAG=1.0
    ```
    

7. Условные команды

- Фишка: Выполняй по условию.
    
    makefile
    
    ```makefile
    check-deps:
        @if ! command -v golangci-lint > /dev/null; then \
            echo "Установи golangci-lint!"; \
            exit 1; \
        fi
    ```
    
    - @ — не выводит команду, \ — многострочник.
        

8. Перехват ошибок

- Фишка: Останавливайся при косяках.
    
    makefile
    
    ```makefile
    build:
        go vet ./... && go build -o myapp ./cmd/main.go
    ```
    
    - && — если vet упал, build не пойдёт.
        

9. Генерация версии

- Фишка: Вставляй версию из Git.
    
    makefile
    
    ```makefile
    VERSION = $(shell git describe --tags --always)
    build:
        go build -ldflags="-X main.Version=$(VERSION)" -o myapp ./cmd/main.go
    ```
    
    - В коде:
        
        go
        
        ```go
        var Version string
        func main() { fmt.Println(Version) }
        ```
        

---

Полный пример с фишками

makefile

```makefile
.PHONY: all build run test clean deps docker fmt lint

APP_NAME = myapp
DOCKER_IMAGE = myuser/$(APP_NAME)
TAG = latest
VERSION = $(shell git describe --tags --always)

all: build

build:
    go vet ./...
    go build -ldflags="-w -s -X main.Version=$(VERSION)" -o $(APP_NAME) ./cmd/main.go

run:
    go run ./cmd/main.go

test:
    go test ./... -v

clean:
    rm -f $(APP_NAME)

deps:
    go mod tidy
    go mod download

fmt:
    go fmt ./...

lint:
    golangci-lint run

docker-build:
    docker build -t $(DOCKER_IMAGE):$(TAG) .

docker-push:
    docker push $(DOCKER_IMAGE):$(TAG)

docker: build docker-build docker-push
```

- Запуск: make docker — соберёт, упакует, запушит.
    

---

Типичные косяки

1. Табуляция:
    
    - Используй Tab, а не пробелы перед командами — make ругнётся.
        
2. Зависимости не указаны:
    
    - build: deps — иначе go build упадёт без модулей.
        
3. Команда не найдена:
    
    - Проверь PATH или установи утилиты (golangci-lint).
        
4. Кэш Git:
    
    - VERSION не обновляется? Делай git tag или git commit.
        

---

Где повторять

- Дока: man make, https://www.gnu.org/software/make/manual/.
    
- Go-специфика: go help build, go test --help.
    
- Практика: Напиши API, добавь test, lint, docker в Makefile.
    

---

Итог

- Основы: build, run, test, clean.
    
- Фишки: Переменные, теги, линтинг, Docker.
    
- Приколы: VERSION из Git, статическая сборка.
    

Запомни как на трассе:

- Makefile — пульт механика.
    
- Цели — кнопки: build, test, docker.
    
- Фишки — турбо для скорости.
    

Ты в деле с Go — давай заюзаем это в твоём проекте? Гони дальше! ![😎](https://abs-0.twimg.com/emoji/v2/svg/1f60e.svg "Smiling face with sunglasses")