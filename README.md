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
# Проверка установленной версии Docker
docker -- version
docker ps
```
<img width="950" height="250" alt="image" src="https://github.com/user-attachments/assets/26095e63-e182-44e6-87c0-e4d45c66d68e" />
```bash
# Создание директории для практики
mkdir docker-practice
cd docker-practice
```
<img width="517" height="56" alt="image" src="https://github.com/user-attachments/assets/9cb901eb-4bd6-4382-8894-ae58cf56c8d2" />
```bash
# Копирование Dockerfile и entrypoint.sh в директорию
# (файлы должны быть созданы согласно примерам выше)
```
<img width="494" height="46" alt="image" src="https://github.com/user-attachments/assets/7ec193a2-cc91-4640-95d9-a975249a100f" />
<img width="879" height="650" alt="image" src="https://github.com/user-attachments/assets/1705e242-681b-4b42-b314-3a45959a35a3" />
<img width="872" height="643" alt="image" src="https://github.com/user-attachments/assets/3c5cf0c5-cd30-45fd-b09c-efa08d89ca5b" />
```bash
#Выдаем права
chmod +x entrypoint.sh
```
<img width="558" height="37" alt="image" src="https://github.com/user-attachments/assets/053c6fa5-90bf-4f0f-88f1-d8eb64b77636" />
```bash
# Сборка образа
docker build -t practice-image:1.0 .
```
<img width="865" height="488" alt="image" src="https://github.com/user-attachments/assets/5e61e363-ea5e-4412-af6e-5a1dae629dd2" />
<img width="888" height="83" alt="image" src="https://github.com/user-attachments/assets/39da7a90-bf34-44cb-aa44-e32f50de517c" />



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
<img width="893" height="576" alt="image" src="https://github.com/user-attachments/assets/04aaf4c7-464e-464e-8829-9b5947343c0b" />



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
<img width="888" height="199" alt="image" src="https://github.com/user-attachments/assets/31e6c5b4-3a3a-4f04-9852-48b950a35dae" />



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
<img width="893" height="237" alt="image" src="https://github.com/user-attachments/assets/216e71d0-184a-4efe-a8cd-b8489b9064eb" />



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
<img width="880" height="515" alt="image" src="https://github.com/user-attachments/assets/11eed9b3-21e4-43ac-bc34-eb27cd62db46" />



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
<img width="874" height="708" alt="image" src="https://github.com/user-attachments/assets/b3305642-4ad4-45a8-91ed-5c63c68972f3" />


---

## Вывод

В ходе практической работы были изучены дополнительные возможности Docker для управления контейнерами. В процессе выполнения заданий я научился получать и сохранять логи контейнеров, мониторить использование системных ресурсов с помощью команды docker stats, а также ограничивать контейнеры по CPU и оперативной памяти. Кроме того, был рассмотрен механизм экспорта контейнера в tar-архив и его последующего импорта с восстановлением работоспособного контейнера.
