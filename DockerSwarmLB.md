# Docker Swarm ile Load Balancer Konfigürasyonu

## 1. Docker Kurulumu

```bash
sudo apt-get update -y && sudo apt-get upgrade -y
apt install docker.io
sudo systemctl status docker
sudo systemctl enable docker
```
## 2. Docker Compose Kurulumu

```bash
sudo curl -L "https://github.com/docker/compose/releases/download/v2.22.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version
```
## 3. Docker Swarm Init ve Master Node Oluşturma
```bash
sudo docker swarm init --advertise-addr <MASTER_NODE_IP_ADDRESS>
```
## 4. Worker Node'ları Cluster'a Dahil Etme
```bash
sudo docker swarm join --token <TOKEN> <MANAGER_IP>:2377
```
## 5. Docker Compose Dosyası Oluşturma
```bash
sudo nano docker-compose.yml
```
### Yaml File
```bash
version: '3.8'
services:
  randomapp:
    image: samyoune/randomapp-v1.2
    deploy:
      mode: replicated
      replicas: 10
      update_config:
        parallelism: 2
        delay: 10s
      restart_policy:
        condition: on-failure
    networks:
      - app-network
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:90"]
      interval: 30s
      timeout: 10s
      retries: 3

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    deploy:
      placement:
        constraints:
          - node.role == manager
    depends_on:
      - randomapp
    networks:
      - app-network

networks:
  app-network:
    driver: overlay
  ```
## 6. Alternatif Docker Compose Dosyası
### Yaml File
```bash
version: '3.8'
services:
  randomapp:
    image: samyoune/randomapp-v1.2
    deploy:
      mode: replicated
      replicas: 18
      placement:
        constraints:
          - node.role == worker
      update_config:
        parallelism: 2
        delay: 10s
        order: start-first
      restart_policy:
        condition: any
        delay: 5s
        max_attempts: 3
        window: 120s
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  nginx:
    image: nginx:latest
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
    deploy:
      mode: replicated
      replicas: 2
      placement:
        constraints:
          - node.role == manager
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: any
        delay: 5s

networks:
  app-network:
    driver: overlay
    attachable: true
  ```
## 7. Nginx Config Dosyasını Oluşturma
```bash
sudo nano nginx.conf
```
### Config File
```bash
events {
    worker_connections 1024;
}

http {
    upstream randomapp_backend {
        server randomapp:90;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://randomapp_backend;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        }
    }
}
```
## 8. Alternatif Nginx Config Dosyası
### Config File
```bash
events {
    worker_connections 4096;
    multi_accept on;
    use epoll;
}

http {
    upstream randomapp_backend {
        least_conn;
        server randomapp:90;
        keepalive 32;
    }

    server {
        listen 80;
        
        location / {
            proxy_pass http://randomapp_backend;
            proxy_http_version 1.1;
            proxy_set_header Connection "";
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
            
            proxy_buffering on;
            proxy_buffer_size 4k;
            proxy_buffers 8 4k;
        }
        
        location /health {
            access_log off;
            return 200 "healthy\n";
        }
    }
}
```
## 9. Stack'i Deploy Etme
```bash
docker stack deploy -c docker-compose.yml randomapp-stack
```
## 10. Servisleri Listeleme
```bash
docker service ls
```
## 11. randomapp Servisinin Çalışan Container'larını Görüntüleme
```bash
docker service ps randomapp-stack_randomapp
```
## 12. Test için Container İçerisinden Nginx'e HTTP İsteği Gönderme
```bash
curl http://[SWARM_IP]
```
## 13. Servis Loglarını Görüntüleme
```bash
docker service logs randomapp-stack_randomapp
```
## Notlar
Docker Swarm cluster'ındaki tüm node'larda Docker Engine yüklü olmalıdır. Swarm, Docker’ın kendi clustering özelliğidir, bu nedenle her bir node üzerinde Docker kurulu olması yeterlidir. Docker Swarm, Docker Engine ile birlikte gelir; yani Docker’ı kurduğunuzda Swarm özellikleri de otomatik olarak sağlanır.
