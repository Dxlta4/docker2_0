# Практическая работа: Дополнительные возможности Docker

## Цель работы

Изучение дополнительных возможностей Docker для управления жизненным циклом контейнеров, мониторинга их состояния и оптимизации использования системных ресурсов. Данная практика направлена на глубокое понимание операционных аспектов контейнеризации, что критически важно для построения надежных и масштабируемых приложений в production-среде.

---

## Dockerfile для практической работы

```dockerfile
# Используется базовый образ на основе Alpine Linux для минимизации размера
FROM alpine:3.18

# Установка необходимых утилит для работы практики
RUN apk add --no-cache \
    bash \
    curl \
    htop \
    procps

# Создание пользователя без привилегий администратора для повышения безопасности
RUN addgroup -g 1000 appuser && \
    adduser -D -u 1000 -G appuser appuser

# Установка рабочей директории
WORKDIR /home/appuser

# Копирование скрипта логирования в контейнер
COPY --chown=appuser:appuser entrypoint.sh /home/appuser/

# Предоставление прав на выполнение скрипта
RUN chmod +x /home/appuser/entrypoint.sh

# Переключение на непривилегированного пользователя
USER appuser

# Точка входа контейнера - выполняет логирование и мониторинг
ENTRYPOINT ["/home/appuser/entrypoint.sh"]

# Значение по умолчанию - время работы в секундах
CMD ["60"]
```

### Скрипт entrypoint.sh

```bash
#!/bin/bash

DURATION=${1:-60}

echo "====== Docker Practice Container ======"
echo "Start time: $(date)"
echo "Duration: ${DURATION} seconds"
echo "========================================"

# Вывод информации о системе
echo "System Information:"
uname -a
echo "Available Memory: $(free -h | grep Mem)"
echo "CPU Info:"
nproc

# Генерируем логи и нагрузку на ресурсы
counter=0
while [ $counter -lt $DURATION ]; do
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Iteration $((counter + 1))/$DURATION"
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] Memory usage: $(ps aux | awk '{sum+=$6} END {print sum/1024 " MB"}')"
    
    # Имитация работы приложения
    sleep 5
    counter=$((counter + 5))
done

echo "====== Container Shutdown ======"
echo "End time: $(date)"
echo "Container completed successfully"
```
<img width="1093" height="200" alt="Снимок экрана 2025-12-23 в 22 50 14" src="https://github.com/user-attachments/assets/832484ca-64a0-4c87-9559-0cbc91935205" />

---

##  1: Подготовка окружения

```bash
# Создание директории для практики
mkdir docker-practice
cd docker-practice

# Копирование Dockerfile и entrypoint.sh в директорию
# (файлы должны быть созданы согласно примерам выше)

# Сборка образа
docker build -t practice-image:1.0 .
```

### Задание 1: Вывод логов в файл

```bash
# Запуск контейнера
docker run -d --name practice-container-1 practice-image:1.0 30

# Ожидание завершения контейнера
sleep 35

# Сохранение логов
docker logs practice-container-1 > /tmp/container_logs.txt

# Просмотр логов
cat /tmp/container_logs.txt

# Очистка
docker rm practice-container-1
```

### Задание 2: Проверка docker-stats

```bash
# Запуск контейнера в background
docker run -d --name practice-container-2 practice-image:1.0 45

# В другом терминале: просмотр статистики в реальном времени
docker stats practice-container-2

# Или сохранение одного снимка статистики
docker stats --no-stream practice-container-2 > /tmp/container_stats.txt

# После завершения контейнера
docker rm practice-container-2
```

### Задание 3: Ограничение ресурсов

```bash
# Запуск контейнера с ограничением памяти и CPU
docker run -d --name practice-limited \
  --memory=256m \
  --cpus=0.5 \
  practice-image:1.0 60

# Проверка статистики с учетом лимитов
docker stats --no-stream practice-limited

# Обновление лимитов на работающем контейнере
docker update --memory=512m practice-limited

# Очистка
docker stop practice-limited
docker rm practice-limited
```

### Задание 4: Экспорт в tar

```bash
# Запуск контейнера и ожидание его завершения
docker run -d --name practice-export practice-image:1.0 30

# Ожидание завершения
sleep 35

# Экспорт контейнера в tar
docker export practice-export > /tmp/container_export.tar

# Проверка размера архива
ls -lh /tmp/container_export.tar

# Просмотр содержимого архива
tar -tf /tmp/container_export.tar | head -20

# Очистка
docker rm practice-export
```

### Задание 5: Импорт из tar

```bash
# Загрузка образа из архива
docker import /tmp/container_export.tar restored-practice:1.0

# Проверка наличия образа
docker images | grep restored

# Запуск контейнера из загруженного образа
docker run -d --name restored-from-tar restored-practice:1.0 /home/appuser/entrypoint.sh

# Проверка логов восстановленного контейнера
sleep 35
docker logs restored-from-tar

# Очистка
docker stop restored-from-tar
docker rm restored-from-tar
docker rmi restored-practice:1.0
```

---

## Контрольные вопросы для самопроверки

1. Какие потоки данных перехватывает Docker при логировании и почему это важно для диагностики?
2. Каким образом контрольные группы (cgroups) Linux обеспечивают ограничение ресурсов в Docker?
3. Чем отличается экспорт контейнера от сохранения образа, и в каких сценариях применяется каждый подход?
4. Почему важно устанавливать ограничения памяти для контейнеров в многоконтейнерной среде?
5. Какие метрики предоставляет `docker stats` и как их интерпретировать для оптимизации производительности?
6. Как восстановить контейнер из tar-архива и какие ограничения существуют при этом процессе?
7. Каким образом можно использовать экспорт контейнеров для миграции приложений между хостами?
