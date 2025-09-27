У меня есть несколько прокси для фермы, и я подумал: почему бы мне не использовать их в Datagram. Используется два скрипта. Основной устанавливает ноды с использованием ваших ключей и прокси. Дополнительный мониторит через заданные промежутки времени (используем crontab) работоспособность всех нод и восстанавливает их в случае необходимости. Второй скрипт особенно актуален из-за того, что прокси также имеют свойство отваливаться.

   Если у вас до этого уже стоял Datagram, то я бы удалил.
Завершим все datagram-сессии:


screen -ls | grep datagram | awk '{print $1}' | xargs -I{} screen -S {} -X quit

проверь, что ничего не осталось:

screen -ls

Дальше можно удалить все папки и файлы, которые относятся к Datagram.

   Переходим к установке основного скрипта.

Сохрани скрипт в файл:


nano install_datagram_nodes.sh

Вставь скрипт, укажи свои ключи и прокси в массивах KEYS и PROXIES. Обрати внимание, что количество нод (NODE_COUNT) можно сделать любым, но оно должно совпадать с количеством ключей (в примере всех по 5) и количеством строк для прокси (первая нода без прокси):

#!/bin/bash

NODE_COUNT=5
INSTALL_DIR="/opt/datagram"
CLI_PATH="$INSTALL_DIR/datagram-cli"

KEYS=(
    "key_for_node_1"
    "key_for_node_2"
    "key_for_node_3"
    "key_for_node_4"
    "key_for_node_5"
)

PROXIES=(
    ""  # Node 1 — без прокси
    "http://user2:pass3@proxy3.com:8080"
    "http://user3:pass3@proxy3.com:8080"
    "http://user4:pass4@proxy4.com:8080"
    "http://user5:pass5@proxy5.com:8080"
)

if [ ${#KEYS[@]} -ne $NODE_COUNT ] || [ ${#PROXIES[@]} -ne $NODE_COUNT ]; then
    echo "❌ Кол-во ключей/прокси не соответствует NODE_COUNT"
    exit 1
fi

mkdir -p "$INSTALL_DIR"

# Скачиваем CLI один раз
if [ ! -f "$CLI_PATH" ]; then
    echo "⬇️ Скачиваем datagram-cli..."
    wget -O "$CLI_PATH" https://github.com/Datagram-Group/datagram-cli-release/releases/latest/download/datagram-cli-x86_64-linux
    chmod +x "$CLI_PATH"
fi

# Запуск нод
for ((i=0; i<NODE_COUNT; i++)); do
    NODE_ID=$((i+1))
    SESSION_NAME="datagram$NODE_ID"
    NODE_DIR="$INSTALL_DIR/node$NODE_ID"
    mkdir -p "$NODE_DIR"

    KEY="${KEYS[$i]}"
    PROXY="${PROXIES[$i]}"

    echo "$KEY" > "$NODE_DIR/key.txt"
    if [ -n "$PROXY" ]; then
        echo "$PROXY" > "$NODE_DIR/proxy.txt"
    fi

    CMD="HOME=$NODE_DIR $CLI_PATH run -- -key $KEY"

    if [ -n "$PROXY" ]; then
        screen -dmS "$SESSION_NAME" bash -c "HTTP_PROXY='$PROXY' HTTPS_PROXY='$PROXY' $CMD"
        echo "✅ Нода $NODE_ID запущена с прокси"
    else
        screen -dmS "$SESSION_NAME" bash -c "$CMD"
        echo "✅ Нода $NODE_ID запущена без прокси"
    fi
done

 Все ноды работают на одном порту. Такое возможно благодаря наличию прокси.


   Переходим к вспомогательному скрипту. Он проверяет: не отвалилась ли какая нода и восстанавливает неработающие. Для автозапуска используется crontab, для проверки работоспособности прокси используется netcat, который установится, если его нет. Делаем второй скрипт: 


nano check_datagram_nodes.sh

Вставляем скрипт. Обрати внимание, что NODE_COUNT должен соответствовать значению из первого скрипта:


#!/bin/bash

NODE_COUNT=5
INSTALL_DIR="/opt/datagram"
CLI_NAME="datagram-cli"
CLI_PATH="$INSTALL_DIR/$CLI_NAME"

# Устанавливаем netcat, если не установлен
if ! command -v nc &>/dev/null; then
    echo "[INFO] Устанавливаем netcat..."
    apt update && apt install -y netcat
fi

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $*"
}

for ((i=1; i<=NODE_COUNT; i++)); do
    NODE_DIR="$INSTALL_DIR/node$i"
    SESSION_NAME="datagram$i"

    if [ ! -d "$NODE_DIR" ]; then
        log "❌ Папка $NODE_DIR не найдена — пропускаем"
        continue
    fi

    KEY_FILE="$NODE_DIR/key.txt"
    if [ ! -f "$KEY_FILE" ]; then
        log "❌ Нет key.txt для node$i — пропускаем"
        continue
    fi
    KEY=$(<"$KEY_FILE")

    PROXY_FILE="$NODE_DIR/proxy.txt"
    PROXY=""
    if [ -f "$PROXY_FILE" ]; then
        PROXY=$(<"$PROXY_FILE")
    fi

    if screen -list | grep -q "$SESSION_NAME"; then
        log "✅ Нода $i (screen: $SESSION_NAME) работает"
        continue
    fi

    PORT=$((5050 + i))
    if nc -z 127.0.0.1 "$PORT"; then
        log "✅ Нода $i отвечает на порт $PORT, но screen не найден"
    else
        log "⚠️ Нода $i не работает — перезапускаем..."

        cd "$NODE_DIR"
        cp "$CLI_PATH" ./$CLI_NAME

        CMD="HOME=$NODE_DIR ./$CLI_NAME run -- -key $KEY"

        if [ -n "$PROXY" ]; then
            screen -dmS "$SESSION_NAME" bash -c "HTTP_PROXY='$PROXY' HTTPS_PROXY='$PROXY' $CMD"
            log "🔁 Нода $i запущена через прокси"
        else
            screen -dmS "$SESSION_NAME" bash -c "$CMD"
            log "🔁 Нода $i запущена без прокси"
        fi
    fi
done

Дополнительный скрипт можно спокойно запускать даже когда у вас уже работают ноды после первого скрипта - он не будет их дублировать.

   Если crontab нет, то его нужно установить:


apt update
apt install cron -y

systemctl enable cron
systemctl start cron

✅ Дальнейшие шаги:
Установка всех нод:


chmod +x /root/install_datagram_nodes.sh
/root/install_datagram_nodes.sh

Мониторинг и перезапуск упавших:


chmod +x /root/check_datagram_nodes.sh
/root/check_datagram_nodes.sh

Заходим в cron:


crontab -e

Добавить последней строкой и сохраниться в crontab (здесь проверка каждые 10 минут):


*/10 * * * * /root/check_datagram_nodes.sh >> /var/log/datagram-check.log 2>&1

Проверь screen-сессии:


screen -ls

Проверь статус на сайте: https://client.datagram.group/

Команды, которые могут пригодиться:
Завершить одну сессию (здесь 3-я):


screen -S datagram3 -X quit# Datagram--
установка нескольких нод на одном сервере с использованием прокси
