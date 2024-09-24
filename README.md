#!/bin/bash

# Log dosyasını oluştur
LOG_FILE="/tmp/screen_log.txt"
DOCKER_LOG_FILE="/tmp/docker_run.log"
CONTAINER_NAME="nillion_accuser"
touch $LOG_FILE

# RPC endpoint listesi
RPC_ENDPOINTS=(
    "65.108.125.39:55657"
    "65.109.124.86:49657"
    "65.108.105.8:22657"
    "65.109.70.228:40657"
    "65.108.79.38:49657"
    "116.202.217.20:36657"
    "65.109.19.204:35657"
    "95.216.246.20:19657"
    "65.21.205.217:23657"
    "65.21.202.101:15657"
    "162.55.98.31:27657"
    "62.112.10.13:25000"
    "136.243.9.249:44657"
    "176.9.120.197:25657"
    "195.201.12.22:32657"
    "78.46.61.108:11657"
    "65.108.124.102:51657"
    "144.76.73.118:16657"
    "213.199.55.13:26657"
    "116.202.82.233:44657"
    "78.46.76.145:32657"
    "65.109.32.239:10657"
    "37.27.69.161:22657"
    "65.109.21.150:45657"
    "159.69.61.113:26657"
    "116.202.174.53:12657"
    "167.235.115.23:28157"
    "88.198.7.204:54657"
    "65.109.228.73:26657"
    "65.109.60.108:19657"
    "65.21.239.41:39657"
    "65.108.131.160:21657"
    "65.109.115.62:10657"
    "5.9.148.136:60657"
    "65.109.75.155:24757"
    "157.90.33.254:44657"
    "95.217.37.139:31657"
    "65.109.80.168:18657"
    "193.87.163.57:26657"
    "129.213.47.133:26657"
    "37.27.69.160:59657"
    "37.27.114.99:29657"
    "51.89.195.146:26657"
    "65.109.30.106:28657"
    "65.21.230.12:11657"
    "109.199.121.58:26657"
)


# Rastgele bir RPC adresi seçme fonksiyonu
select_random_rpc() {
    local index=$((RANDOM % ${#RPC_ENDPOINTS[@]}))
    echo ${RPC_ENDPOINTS[$index]}
}

# RPC adreslerinden block yüksekliğini alma fonksiyonu
get_latest_block_height() {
    for i in {1..5}; do  # Maksimum 5 deneme
        RPC_ENDPOINT=$(select_random_rpc)

        # Block yüksekliğini alma denemesi
        HEIGHT=$(curl -s http://$RPC_ENDPOINT/status | jq -r .result.sync_info.latest_block_height)

        # Eğer yükseklik yalnızca sayı içeriyorsa döndür
        if [[ $HEIGHT =~ ^[0-9]+$ ]]; then
            echo $HEIGHT
            return 0
        else
            echo "Hata: $RPC_ENDPOINT adresi ile block yüksekliği alınamadı, başka RPC adresi deneniyor..." >&2
        fi
    done

    echo "Başarısız: Hiçbir RPC adresi block yüksekliğini alamadı." >&2
    exit 1
}

# Docker konteynerinin çalışıp çalışmadığını kontrol etme fonksiyonu
check_and_start_docker() {
    if [ "$(docker ps -q -f name=$CONTAINER_NAME)" ]; then
        echo "Docker konteyneri zaten çalışıyor."
    else
        echo "Docker konteyneri çalışmıyor, başlatılıyor..."
        RPC_ENDPOINT=$(select_random_rpc)
        BLOCK_START=$(get_latest_block_height)
        docker run -d --name $CONTAINER_NAME -v ./nillion/accuser:/var/tmp nillion/verifier:v1.0.0 verify --rpc-endpoint "http://$RPC_ENDPOINT" --block-start $BLOCK_START > $DOCKER_LOG_FILE 2>&1

        # Docker komutunun çıkış durumunu kontrol et
        DOCKER_EXIT_CODE=$?
        if [ $DOCKER_EXIT_CODE -ne 0 ]; then
            echo "Docker konteyneri başlatılamadı. Hata oluştu. Çıkış Kodu: $DOCKER_EXIT_CODE"
            tail -n 20 $DOCKER_LOG_FILE  # Son 20 satırı göster
        else
            echo "Docker konteyneri başarıyla başlatıldı."
        fi
    fi
}

# Ana script
while true; do
    echo "Kontrol ediliyor..."
    # Block yüksekliğini al
    BLOCK_START=$(get_latest_block_height)

    # Log dosyasını kontrol et ve çıktısını göster
    echo "Log dosyası içeriği:"
    cat $LOG_FILE

    # Eğer bir hata tespit edilirse node'u yeniden başlat
    if grep -q "Connection refused (os error 111)" $LOG_FILE; then
        echo "Error detected, restarting Nillion accuser node..."
        docker stop $CONTAINER_NAME && docker rm $CONTAINER_NAME
        check_and_start_docker

        # Log dosyasını temizle
        > $LOG_FILE
    else
        echo "Herhangi bir hata tespit edilmedi."
        # Docker konteyneri çalışıyor mu kontrol et, çalışmıyorsa başlat
        check_and_start_docker
    fi
    sleep 10  # Her 10 saniyede bir kontrol
done
