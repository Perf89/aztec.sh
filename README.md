#!/bin/bash

# Проверка и установка нужных утилит
for util in figlet whiptail curl docker iptables jq; do
  if ! command -v "$util" &>/dev/null; then
    echo "$util не найден. Устанавливаю..."
    sudo apt update && sudo apt install -y "$util"
  fi
done

# Цвета
GREEN="\e[32m"
YELLOW="\e[33m"
CYAN="\e[36m"
RED="\e[31m"
PURPLE="\e[35m"
NC="\e[0m"

# Брендинг
echo -e "\n\n"
echo -e "${CYAN}$(figlet -w 150 -f standard "Soft by Perfnode")${NC}"
echo "========================================================================"
echo "     Добро пожаловать в мастер установки ноды Aztec от Perfnode         "
echo "========================================================================"

echo -e "${YELLOW}Telegram канал: https://t.me/perfnode${NC}"
echo -e "${CYAN}Продолжение через 10 секунд...${NC}"
sleep 10

# Анимация
animate() {
  for i in {1..3}; do
    printf "\r${GREEN}Загрузка${NC}%s" "$(printf '.%.0s' $(seq 1 $i))"
    sleep 0.4
  done
  echo
}
animate

# Благодарность
give_ack() {
  echo
  echo -e "${CYAN}Спасибо! Подписывайтесь: https://t.me/perfnode${NC}"
}

# Меню
while true; do
CHOICE=$(whiptail --title "Меню Aztec Node - Perfnode" --menu "Выберите действие:" 20 70 10 \
"1" "Установить ноду" \
"2" "Обновить ноду" \
"3" "Показать логи" \
"4" "Проверить хеш" \
"5" "Зарегистрировать валидатора" \
"6" "Удалить ноду" \
"7" "Выйти из скрипта" 3>&1 1>&2 2>&3)

case $CHOICE in
  1)
    read -p "Введите URL RPC Sepolia: " RPC
    read -p "Введите URL Beacon Sepolia: " CONSENSUS
    read -p "Введите адрес кошелька (0x...): " WALLET
    read -p "Введите приватный ключ кошелька (0x...): " PRIVATE_KEY
    SERVER_IP=$(curl -s https://api.ipify.org)

    sudo apt update && sudo apt upgrade -y
    sudo apt install -y curl git unzip lz4 jq make gcc build-essential tmux wget ncdu \
    iptables-persistent figlet whiptail libssl-dev pkg-config libgbm1 docker.io

    sudo systemctl enable docker
    sudo systemctl start docker
    sudo usermod -aG docker $USER
    sudo iptables -I INPUT -p tcp --dport 40400 -j ACCEPT
    sudo iptables -I INPUT -p udp --dport 40400 -j ACCEPT
    sudo iptables -I INPUT -p tcp --dport 8080 -j ACCEPT
    sudo sh -c "iptables-save > /etc/iptables/rules.v4"

    mkdir -p "$HOME/aztec-sequencer/data"
    cd "$HOME/aztec-sequencer"
    docker pull aztecprotocol/aztec:0.87.2

    cat > .env <<EOF
ETHEREUM_HOSTS=$RPC
L1_CONSENSUS_HOST_URLS=$CONSENSUS
VALIDATOR_PRIVATE_KEY=$PRIVATE_KEY
P2P_IP=$SERVER_IP
WALLET=$WALLET
GOVERNANCE_PROPOSER_PAYLOAD_ADDRESS=0x54F7fe24E349993b363A5Fa1bccdAe2589D5E5Ef
EOF

    docker run -d \
      --name aztec-sequencer \
      --network host \
      --entrypoint /bin/sh \
      --env-file "$HOME/aztec-sequencer/.env" \
      -e DATA_DIRECTORY=/data \
      -e LOG_LEVEL=debug \
      -v "$HOME/aztec-sequencer/data":/data \
      aztecprotocol/aztec:0.87.2 \
      -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --node --archiver --sequencer'

    echo -e "${YELLOW}Установка завершена.${NC}"
    docker logs --tail 100 -f aztec-sequencer
    ;;
  2)
    docker pull aztecprotocol/aztec:0.87.2
    docker stop aztec-sequencer
    docker rm aztec-sequencer
    docker run -d \
      --name aztec-sequencer \
      --network host \
      --entrypoint /bin/sh \
      --env-file "$HOME/aztec-sequencer/.env" \
      -e DATA_DIRECTORY=/data \
      -e LOG_LEVEL=debug \
      -v "$HOME/aztec-sequencer/data":/data \
      aztecprotocol/aztec:0.87.2 \
      -c 'node --no-warnings /usr/src/yarn-project/aztec/dest/bin/index.js start --network alpha-testnet --node --archiver --sequencer'
    ;;
  3)
    docker logs --tail 100 -f aztec-sequencer
    ;;
  4)
    TIP=$(curl -s -X POST -H "Content-Type: application/json" \
      -d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":1}' http://localhost:8080)
    BLK=$(echo "$TIP" | jq -r '.result.proven.number')
    PROOF=$(curl -s -X POST -H "Content-Type: application/json" \
      -d "{\"jsonrpc\":\"2.0\",\"method\":\"node_getArchiveSiblingPath\",\"params\":[${BLK},${BLK}],\"id\":1}" http://localhost:8080 | jq -r '.result')
    echo -e "${GREEN}Блок:${NC} $BLK"
    echo -e "${GREEN}Хеш:${NC} $PROOF"
    ;;
  5)
    echo -e "${CYAN}Регистрация валидатора...${NC}"
    cd "$HOME/aztec-sequencer"
    [ -f .env ] && export $(grep -v '^#' .env | xargs)
    tmpnv=$(mktemp)
    curl -fsSL https://raw.githubusercontent.com/TheGentIeman/Nodes/refs/heads/main/NewValidator.sh > "$tmpnv"
    chmod +x "$tmpnv"
    bash "$tmpnv"
    rm -f "$tmpnv"
    ;;
  6)
    docker stop aztec-sequencer
    docker rm aztec-sequencer
    rm -rf "$HOME/aztec-sequencer"
    echo -e "${GREEN}Нода удалена.${NC}"
    ;;
  7)
    echo -e "${CYAN}Выход из скрипта.${NC}"
    exit 0
    ;;
esac
done
