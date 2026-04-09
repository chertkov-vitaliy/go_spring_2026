# Лабораторная работа: Создание Go-приложения в Docker, публикация в Docker Hub и развертывание Pod в Kubernetes

## Цель работы
Научиться создавать собственное Go-приложение, упаковывать его в Docker-образ, публиковать в реестре Docker Hub, писать манифест для Kubernetes и управлять Pod с помощью `kubectl`.

## Продолжительность
~40–50 минут

## Необходимое ПО
- Docker
- kubectl
- K8S

## Подготовка

Убедитесь, что кластер работает:
```bash
kubectl cluster-info
```

---

## Часть I. Создание Go-приложения

### Шаг 1. Создание директории и файла main.go

```bash
mkdir go-web-app
cd go-web-app
```

Создайте файл `main.go`:
```bash
nano main.go
```

**Содержимое `main.go`:**
```go
package main

import (
    "fmt"
    "net/http"
    "os"
)

func main() {
    http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
        message := os.Getenv("MESSAGE")
        if message == "" {
            message = "Hello from Go in Kubernetes!"
        }
        fmt.Fprintf(w, "<h1>%s</h1>\n", message)
        fmt.Fprintf(w, "<p>Pod host: %s</p>", r.Host)
    })

    http.HandleFunc("/health", func(w http.ResponseWriter, r *http.Request) {
        w.WriteHeader(http.StatusOK)
        w.Write([]byte("OK"))
    })

    fmt.Println("Server started on :80")
    http.ListenAndServe(":80", nil)
}
```

### Шаг 2 Написание Dockerfile

Создайте `Dockerfile`:
```bash
vim Dockerfile
```

**Содержимое Dockerfile:**
```dockerfile
# Этап сборки (builder)
FROM golang:1.21-alpine AS builder

WORKDIR /app
COPY main.go .

# Сборка статического бинарника
RUN go build -o server main.go

# Финальный образ
FROM alpine:latest

WORKDIR /root/
COPY --from=builder /app/server .

EXPOSE 80

CMD ["./server"]
```

> **Примечание:** Многоэтапная сборка позволяет получить минимальный образ (~15 МБ вместо ~800 МБ).

---

## Часть II. Сборка и публикация Docker-образа

### Шаг 1. Сборка образа

```bash
docker build -t yourusername/go-web-app:latest .
```

> **Важно:** замените `yourusername` на ваш реальный логин в Docker Hub.

Просмотр собранного образа 
```bash
docker rmi
```

### Шаг 2. Локальное тестирование

```bash
docker run -p 8888:80 yourusername/go-web-app:latest
```

Откройте в браузере: [http://localhost:8888](http://localhost:8888)

Вы должны увидеть приветствие от Go-сервера.

Остановите контейнер: `Ctrl+C`

### Шаг 3 Публикация в Docker Hub

```bash
docker login
docker push yourusername/go-web-app:latest
```

Убедитесь, что образ появился на [hub.docker.com](https://hub.docker.com) в вашем репозитории.

---

## Часть III. Написание манифеста для Kubernetes

### Шаг 3.1 Создание файла манифеста

```bash
vim mypod.yaml
```

**Содержимое `mypod.yaml`:**
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: go-web-pod
  labels:
    app: go-web-app
spec:
  containers:
  - name: go-app
    image: yourusername/go-web-app:latest
    ports:
    - containerPort: 80
    env:
    - name: MESSAGE
      value: "Custom message from K8s manifest!"
    # livenessProbe:
    #   httpGet:
    #     path: /health
    #     port: 80
    #   initialDelaySeconds: 5
    #   periodSeconds: 10
```

> **Примечание:** Раскомментируйте `livenessProbe` для дополнительного задания.

---

## Часть IV. Развертывание Pod в Kubernetes

### Шаг 1 Создание Pod

```bash
kubectl apply -f mypod.yaml
```

### Шаг 2 Проверка статуса

```bash
kubectl get pods
```

Ожидаемый вывод: `STATUS: Running`

### Шаг 3 Детальная информация

```bash
kubectl describe pods go-web-pod
```

### Шаг 4 Доступ к приложению

Пробросьте порт:
```bash
kubectl port-forward go-web-pod 7777:80
```

Откройте в браузере: [http://localhost:7777](http://localhost:7777)

Вы должны увидеть кастомное сообщение из переменной окружения `MESSAGE`.

---

## Часть V. Управление Pod (работа с командами)

### Шаг 1 Просмотр логов

```bash
kubectl logs go-web-pod
```

### Шаг 2 Выполнение команды внутри контейнера

```bash
kubectl exec go-web-pod -- date
```

### Шаг 3 Интерактивный доступ

```bash
kubectl exec -it go-web-pod -- sh
```

Внутри контейнера можно выполнять команды:
```bash
ls -la
cat /etc/os-release
exit
```

---

## Часть VI. Удаление ресурсов

### Шаг 1 Удаление Pod по манифесту

```bash
kubectl delete -f mypod.yaml
```

### Шаг 2 Проверка удаления

```bash
kubectl get pods
```

### Шаг 3 (Опционально) Удаление образа локально

```bash
docker rmi yourusername/go-web-app:latest
```

---

## Справочник команд kubectl

| Команда | Что делает |
|---------|------------|
| `kubectl get pods` | Показать все Pods |
| `kubectl run hello --generator=run-pod/v1 --image=nginx:latest --port=80` | Создать Pod из образа nginx |
| `kubectl port-forward hello 7777:80` | Пробросить порт 7777 на порт 80 Pod |
| `kubectl describe pods hello` | Показать все данные Pod |
| `kubectl delete pods hello` | Удалить Pod |
| `kubectl logs hello` | Показать логи из Pod |
| `kubectl exec my-web date` | Запустить команду date на Pod |
| `kubectl exec -it my-web bash` | Запустить bash интерактивно |
| `kubectl apply -f myfile.yaml` | Создать объекты из манифеста |
| `kubectl delete -f myfile.yaml` | Удалить объекты по манифесту |

---

## Контрольные вопросы

1. Зачем используется многоэтапная сборка (multi-stage build) в Dockerfile для Go?
2. Какой порт слушает Go-приложение внутри контейнера? Как это связано с `containerPort` в манифесте?
3. Что произойдёт, если запустить `kubectl apply -f mypod.yaml` повторно?
4. Чем отличается `kubectl exec go-web-pod date` от `kubectl exec -it go-web-pod sh`?
5. Для чего нужна `livenessProbe` и как она работает?
6. Как передать переменную окружения в Pod без редактирования манифеста?

---

## Дополнительные задания (для продвинутых)

1. **Добавить эндпоинт `/env`** — вывод всех переменных окружения.
2. **Настроить ConfigMap** — передавать приветственный текст через `configMapKeyRef`.
3. **Активировать livenessProbe** (раскомментировать в манифесте) и проверить её работу.
4. **Собрать образ с тегом `v1`** и загрузить в Docker Hub, затем обновить Pod.
5. **Написать второй манифест** для создания Pod через `kubectl run` с вашим Go-образом:
   ```bash
   kubectl run go-quick --generator=run-pod/v1 --image=yourusername/go-web-app:latest --port=80
   ```


## Примечания

- Если вы используете **Minikube**, после `minikube start` может потребоваться настроить Docker-окружение:
  ```bash
  eval $(minikube docker-env)
  ```
- Для **Kind** или **k3s** дополнительная настройка не требуется.
- Убедитесь, что образ публичный в Docker Hub, иначе K8s не сможет его скачать.

---


