## 1. Что в итоге должно получиться

Мы создадим проект:

```text
hello-go/
├── .github/
│   └── workflows/
│       └── ci.yml
├── greeting/
│   ├── greeting.go
│   └── greeting_test.go
├── .gitignore
├── .dockerignore
├── Dockerfile
├── go.mod
└── main.go
```

После этого схема будет такой:

```text
Код Go
  ↓
GitHub
  ↓
GitHub Actions
  ↓
gofmt → go vet → go test → go build
  ↓
Docker build
  ↓
GHCR
  ↓
ghcr.io/ВАШ_USERNAME/hello-go:latest
```

Идея задания: при `push` в `main` GitHub автоматически проверяет проект и публикует Docker-образ в GitHub Container Registry. 

---

# 2. Что нужно установить

Перед началом желательно иметь:

* **Docker Desktop**
* **Git**
* аккаунт **GitHub**
* любой редактор кода, например VS Code

При этом **Go на Windows устанавливать необязательно**, потому что тестирование проекта можно выполнять внутри Docker-контейнера. 

Проверить Docker:

```powershell
docker --version
```

Проверить Git:

```powershell
git --version
```

Если обе команды показывают версии — всё готово.

---

# 3. Создаём папку проекта

Для Windows PowerShell:

```powershell
cd ~
mkdir hello-go
cd hello-go
```

Теперь создаём необходимые папки:

```powershell
mkdir greeting
mkdir .github
mkdir .github\workflows
```

Получится:

```text
hello-go
├── .github
│   └── workflows
└── greeting
```

---

# 4. Создаём `go.mod`

В корне проекта:

```text
hello-go/go.mod
```

Вставьте:

```go
module hello-go

go 1.23
```

`go.mod` сообщает Go название модуля и версию Go. В нашем проекте модуль называется `hello-go`. 

---

# 5. Создаём программу `greeting.go`

Создайте:

```text
greeting/greeting.go
```

Код:

```go
package greeting

import "fmt"

func Greet(name string) string {
	return fmt.Sprintf("Hello, %s!", name)
}

func SumRange(from, to int) int {
	sum := 0
	for i := from; i <= to; i++ {
		sum += i
	}
	return sum
}
```

Здесь две функции:

### `Greet`

```go
Greet("Docker")
```

вернёт:

```text
Hello, Docker!
```

### `SumRange`

```go
SumRange(1, 10)
```

посчитает:

```text
1 + 2 + 3 + ... + 10 = 55
```

Это соответствует исходному заданию. 

---

# 6. Создаём тесты

Создайте:

```text
greeting/greeting_test.go
```

Вставьте:

```go
package greeting

import "testing"

func TestGreet(t *testing.T) {
	got := Greet("Docker")
	want := "Hello, Docker!"

	if got != want {
		t.Errorf("Greet() = %q, want %q", got, want)
	}
}

func TestSumRange(t *testing.T) {
	got := SumRange(1, 10)
	want := 55

	if got != want {
		t.Errorf("SumRange(1, 10) = %d, want %d", got, want)
	}
}
```

Go автоматически распознаёт файл как файл тестов благодаря окончанию:

```text
_test.go
```

А функции тестов должны иметь вид:

```go
func TestXxx(t *testing.T)
```



---

# 7. Создаём `main.go`

В корне:

```text
main.go
```

Вставьте:

```go
package main

import (
	"fmt"
	"os"
	"runtime"

	"hello-go/greeting"
)

func main() {
	fmt.Println("Hello from Go in Docker! 🐹🐳")
	fmt.Printf("OS: %s\n", runtime.GOOS)
	fmt.Printf("Arch: %s\n", runtime.GOARCH)
	fmt.Println(greeting.Greet("Docker"))
	fmt.Printf("Sum 1..10 = %d\n", greeting.SumRange(1, 10))

	if len(os.Args) > 1 {
		fmt.Println("Аргументы:")
		for i, arg := range os.Args[1:] {
			fmt.Printf("  %d: %s\n", i+1, arg)
		}
	}
}
```

Здесь запускается основная программа и вызываются функции из пакета `greeting`. 

---

# 8. Создаём Dockerfile

В корне проекта:

```text
Dockerfile
```

Вставьте:

```dockerfile
# Этап 1: сборка
FROM golang:1.23-alpine AS builder

WORKDIR /build

# Копируем файлы зависимостей
COPY go.* ./

# Скачиваем зависимости
RUN go mod download

# Копируем исходный код
COPY . .

# Компилируем приложение
RUN CGO_ENABLED=0 GOOS=linux go build -ldflags="-s -w" -o hello-go .

# Этап 2: запуск
FROM alpine:3.20

# Создаём непривилегированного пользователя
RUN adduser -D appuser

USER appuser

WORKDIR /home/appuser

COPY --from=builder /build/hello-go ./hello-go

ENTRYPOINT ["./hello-go"]
```

Здесь используется **multi-stage build**:

```text
golang:1.23-alpine
        ↓
   компиляция Go
        ↓
     binary
        ↓
alpine:3.20
        ↓
   готовый контейнер
```

То есть компилятор Go не попадает в финальный образ. 

---

# 9. Создаём `.gitignore`

Файл:

```text
.gitignore
```

Содержимое:

```gitignore
/hello-go
.env
.idea/
.vscode/
*.iml
```

---

# 10. Создаём `.dockerignore`

Файл:

```text
.dockerignore
```

Содержимое:

```text
.git/
.github/
*.md
.gitignore
.dockerignore
hello-go
```

Это говорит Docker, какие файлы не нужно отправлять в build context. 

---

# 11. Проверяем структуру

В итоге должна быть такая структура:

```text
hello-go/
│
├── .github/
│   └── workflows/
│       └── ci.yml
│
├── greeting/
│   ├── greeting.go
│   └── greeting_test.go
│
├── .gitignore
├── .dockerignore
├── Dockerfile
├── go.mod
└── main.go
```

---

# 12. Проверяем Go через Docker

Поскольку Go на компьютере нам не нужен, выполняем тесты через Docker.

В PowerShell:

```powershell
cd ~/hello-go
```

Затем:

```powershell
docker run --rm `
  -e GOPATH=/tmp/go `
  -e GOCACHE=/tmp/go-cache `
  -v "${PWD}:/app" `
  -w /app `
  golang:1.23-alpine `
  go test ./... -v
```

Должно появиться примерно:

```text
=== RUN   TestGreet
--- PASS: TestGreet (0.00s)
=== RUN   TestSumRange
--- PASS: TestSumRange (0.00s)
PASS
ok      hello-go/greeting
```

Это означает, что оба теста прошли. 

---

# 13. Собираем Docker-образ

Теперь:

```powershell
docker build -t hello-go .
```
<img width="1794" height="517" alt="image" src="https://github.com/user-attachments/assets/8c3a20cf-f22c-4780-add0-1fcbeacd123a" />

В конце должно быть что-то вроде:

```text
Successfully built ...
Successfully tagged hello-go:latest
```

Проверить образ:

```powershell
docker images
```

Там должен появиться:

```text
hello-go
```

---

# 14. Запускаем контейнер

```powershell
docker run --rm hello-go
```

Ожидаемый результат:

```text
Hello from Go in Docker! 🐹🐳
OS: linux
Arch: amd64
Hello, Docker!
Sum 1..10 = 55
```



Если это появилось — **Go-приложение и Dockerfile работают**.

---

# 15. Создаём GitHub-репозиторий

Теперь заходим на GitHub.

Создаём новый репозиторий:

```text
hello-go
```

### Важно

Репозиторий создаём **пустым**.

Не ставим галочки:

```text
☐ Add a README file
☐ Add .gitignore
☐ Choose a license
```

Иначе первый `push` может потребовать дополнительного объединения истории. В исходном задании также указано создавать пустой репозиторий. 

---

# 16. Создаём GitHub Actions

Теперь самое важное.

Файл должен находиться здесь:

```text
.github/workflows/ci.yml
```

Создайте `ci.yml` и вставьте:

```yaml
name: Go CI/CD

on:
  push:
    branches: [ main ]
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - uses: actions/checkout@v7

      - name: Set up Go
        uses: actions/setup-go@v5
        with:
          go-version: '1.23'
          cache: true

      - name: Format check
        run: |
          UNFORMATTED=$(gofmt -l .)
          if [ -n "$UNFORMATTED" ]; then
            echo "❌ Следующие файлы не отформатированы:"
            echo "$UNFORMATTED"
            echo "Запустите локально: gofmt -w ."
            exit 1
          fi

      - name: Lint with go vet
        run: go vet ./...

      - name: Run tests
        run: go test ./... -v

      - name: Build release
        run: CGO_ENABLED=0 go build -o hello-go .

      - name: Log in to GHCR
        if: github.event_name == 'push' && github.ref == 'refs/heads/main'
        uses: docker/login-action@v4
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Set up Docker Buildx
        uses: docker/setup-buildx-action@v4

      - name: Extract Docker metadata
        id: meta
        uses: docker/metadata-action@v6
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=sha,prefix=,format=short
            type=raw,value=latest,enable={{is_default_branch}}

      - name: Build and push Docker image
        uses: docker/build-push-action@v7
        with:
          context: .
          push: ${{ github.event_name == 'push' && github.ref == 'refs/heads/main' }}
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
```

Это основной файл CI/CD из задания. 

---

# 17. Что делает `ci.yml`

GitHub Actions будет выполнять:

### Шаг 1 — скачать код

```yaml
- uses: actions/checkout@v7
```

### Шаг 2 — установить Go

```yaml
- name: Set up Go
```

Используется Go `1.23`.

### Шаг 3 — проверить форматирование

```yaml
gofmt -l .
```

### Шаг 4 — проверить код

```yaml
go vet ./...
```

### Шаг 5 — запустить тесты

```yaml
go test ./... -v
```

### Шаг 6 — собрать Go-приложение

```yaml
go build
```

### Шаг 7 — войти в GHCR

```yaml
docker/login-action
```

### Шаг 8 — собрать Docker image

```yaml
docker/build-push-action
```

### Шаг 9 — отправить его в GHCR

Именно поэтому это уже не просто CI, а CI/CD. 

---

# 18. Загружаем проект на GitHub

В PowerShell перейдите в проект:

```powershell
cd ~/hello-go
```

Инициализируем Git:

```powershell
git init
```

Добавляем файлы:

```powershell
git add .
```

Создаём commit:

```powershell
git commit -m "Initial commit: Go app with Docker and CI/CD to GHCR"
```

Создаём ветку `main`:

```powershell
git branch -M main
```

Теперь подключаем GitHub.

Например, если твой GitHub username:

```text
SherKron
```

то:

```powershell
git remote add origin https://github.com/SherKron/hello-go.git
```

Проверяем:

```powershell
git remote -v
```

И отправляем:

```powershell
git push -u origin main
```

Эти действия соответствуют шагу публикации проекта из задания. 

---

# 19. Проверяем GitHub Actions

После:

```powershell
git push -u origin main
```

открываем свой репозиторий на GitHub.

Переходим:

```text
Actions
```

Там должен появиться workflow:

```text
Go CI/CD
```

Нажимаем на него.

Внутри увидишь примерно:

```text
✓ Set up Go
✓ Format check
✓ Lint with go vet
✓ Run tests
✓ Build release
✓ Log in to GHCR
✓ Set up Docker Buildx
✓ Extract Docker metadata
✓ Build and push Docker image
```

<img width="2175" height="895" alt="image" src="https://github.com/user-attachments/assets/c4840f77-518d-4694-819e-a32d3eb9cbf2" />

Если все пункты зелёные — CI/CD отработал успешно.

---

# 20. Проверяем GHCR

После успешного workflow открываем страницу своего GitHub-профиля.

В разделе:

```text
Packages
```

должен появиться:

```text
hello-go
```

GitHub будет хранить Docker-образ примерно как:

```text
ghcr.io/ВАШ_USERNAME/hello-go
```
<img width="764" height="166" alt="image" src="https://github.com/user-attachments/assets/62194fa4-7332-4f7b-a95a-1061c6c35a39" />

Исходное задание указывает, что после выполнения workflow пакет должен появиться в `Packages`. 

---

# 21. Делаем пакет Public

По умолчанию GHCR-пакет может быть приватным.

Открываем:

```text
Packages → hello-go
```

Затем:

```text
Package settings
```

Находим:

```text
Danger Zone
```

→

```text
Change visibility
```

→

```text
Public
```

Подтверждаем изменение.

Это отдельная настройка пакета — публичный GitHub-репозиторий сам по себе не делает GHCR-пакет публичным. 

---

# 22. Проверяем, что образ можно скачать

Теперь можно проверить публикацию.

В PowerShell:

```powershell
docker logout ghcr.io
```

Затем:

```powershell
$GITHUB_USER = Read-Host "Введите ваш GitHub username"
```

Например:

```text
SherKron
```

После этого:

```powershell
docker pull "ghcr.io/$GITHUB_USER/hello-go:latest"
```

Если всё получилось, Docker скачает опубликованный образ.

Запускаем:

```powershell
docker run --rm "ghcr.io/$GITHUB_USER/hello-go"
```

Должно получиться:

```text
Hello from Go in Docker! 🐹🐳
OS: linux
Arch: amd64
Hello, Docker!
Sum 1..10 = 55
```



---

# 23. Что показать преподавателю

Я бы подготовил следующие доказательства выполнения:

### 1. Структура проекта

```text
hello-go/
├── .github/workflows/ci.yml
├── greeting/greeting.go
├── greeting/greeting_test.go
├── Dockerfile
├── go.mod
└── main.go
```

### 2. Успешные тесты

Показать:

```text
PASS
ok hello-go/greeting
```

### 3. Docker build

Показать:

```powershell
docker build -t hello-go .
```

### 4. Работа контейнера

Показать:

```powershell
docker run --rm hello-go
```

и:

```text
Hello from Go in Docker!
OS: linux
Arch: amd64
Hello, Docker!
Sum 1..10 = 55
```

### 5. GitHub Actions

Показать зелёный:

```text
Go CI/CD ✓
```

### 6. GHCR

Показать:

```text
Packages
└── hello-go
```

### 7. Проверку скачивания

Показать:

```powershell
docker pull ghcr.io/USERNAME/hello-go:latest
```

и запуск:

```powershell
docker run --rm ghcr.io/USERNAME/hello-go:latest
```

---
