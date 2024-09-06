# nillion-otamatik-yeniden-baslatma-yeni-rpc-ile

1 -scripti nano ile açılan sayfaya yapıstırp kaydedin 

```
nano nillion_start_node.sh
```


```
#!/bin/bash

# Log dosyasını oluştur
LOG_FILE="/tmp/screen_log.txt"
DOCKER_LOG_FILE="/tmp/docker_run.log"
CONTAINER_NAME="nillion_accuser"
touch $LOG_FILE

# RPC endpoint listesi
RPC_ENDPOINTS=(
    "37.27.69.161:22657"
    "65.21.205.217:23657"
    "65.109.228.73:26657"
    "65.109.30.106:28657"
    "95.216.246.20:19657"
    "65.21.239.41:39657"
    "168.119.9.219:46657"
    "37.27.69.160:59657"
    "65.109.61.219:18057"
    "65.21.230.12:11657"
    "159.69.61.113:26657"
    "78.46.61.108:11657"
    "188.40.85.207:13557"
    "116.202.82.233:44657"
    "167.235.115.23:28157"
    "5.9.148.136:60657"
    "78.46.76.145:32657"
    "88.198.7.204:54657"
    "88.99.137.42:53657"
    "157.90.33.254:44657"
    "195.201.12.22:32657"
    "136.243.9.249:44657"
    "116.202.217.20:36657"
    "213.199.55.13:26657"
    "88.99.208.54:31657"
    "65.109.60.108:19657"
    "95.217.37.139:31657"
    "176.9.103.91:47657"
    "62.112.10.13:25000"
    "116.202.174.53:12657"
    "65.109.32.239:10657"
    "65.21.202.101:15657"
    "37.27.114.99:29657"
    "49.12.171.172:43657"
    "193.87.163.57:26657"
    "65.108.70.78:41657"
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
        docker run -d --name $CONTAINER_NAME -v ./nillion/accuser:/var/tmp nillion/retailtoken-accuser:v1.0.1 accuse --rpc-endpoint "http://$RPC_ENDPOINT" --block-start $BLOCK_START > $DOCKER_LOG_FILE 2>&1

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


```
2-Scriti çalıştırılabilir yapın 

```
chmod +x nillion_start_node.sh

```

3-Scrpti  çalıştırın

```

./nillion_start_node.sh

```


**Yukardaki scrpti screen içinde çalıştırmayı unutmayın**



logları görmek içinse aşağıdaki scrpti kullanacağız screen e gerek yok 

1- nano ile log görme scrptini olsuturun ve kaydedin
```
nano view_docker_logs.sh
```

```
#!/bin/bash

# Logları görmek istediğiniz konteynerin adı veya ID'sini belirleyin
CONTAINER_NAME="nillion/retailtoken-accuser:v1.0.1"

# Konteynerin ID'sini veya ismini al
CONTAINER_ID=$(docker ps --filter "ancestor=$CONTAINER_NAME" --format "{{.ID}}")

# Eğer konteyner çalışmıyorsa uyarı ver
if [ -z "$CONTAINER_ID" ]; then
    echo "Hata: Konteyner bulunamadı veya çalışmıyor. Lütfen doğru konteyner adını kontrol edin."
    exit 1
fi

# Logların son 200 satırını takip et ve ekrana yazdır
echo "Son 200 satır log görüntüleniyor... (CTRL+C ile çıkabilirsiniz)"
docker logs --tail 200 -f $CONTAINER_ID

```
2-Scriti çalıştırılabilir yapın 

```
chmod +x nillion_start_node.sh

```

3-Scrpti  çalıştırın

```

./view_docker_logs.sh

```
