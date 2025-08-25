
---

# **Docker Logging + Nginx + Fluentd **

This project demonstrates:

* A **custom Alpine container** with `cron` writing a timestamp **every minute** to `/var/logs/time.log`.
* **Nginx** serving a local `index.html` on **port 8080**.
* **Fluentd** tailing the shared log and printing to **stdout**.
* A **custom bridge network** for name-based container communication.
* **Persistent logs** via a named Docker volume.
* **CPU/Memory limits** on each container.

---

## **Folder Structure**

```
custom-logger/
  Dockerfile
  cronjob.sh
  crontab.txt
fluentd/
  fluent.conf
html/
  index.html
```

---

## **Setup Instructions**

### **1. Create a Custom Network & Volumes**

```bash
docker network create custom-net
docker volume create logs-vol
```

---

### **2. Build the Custom Logger Image**

```bash
docker build -t custom-logger:latest ./custom-logger
```

---

### **3. Start Containers**

#### **a) Custom Logger (Cron → /var/logs/time.log)**

```bash
docker run -d \
  --name custom-logger \
  --network custom-net \
  -v logs-vol:/var/logs \
  --restart unless-stopped \
  --cpus 0.5 --memory 256m \
  custom-logger:latest
```

---

#### **b) Nginx (Serve HTML on Port 8080)**

Bind-mount local `html` folder:

```bash
docker run -d \
  --name nginx-server \
  --network custom-net \
  -p 8080:80 \
  -v "$(pwd)"/html:/usr/share/nginx/html:ro \
  --restart unless-stopped \
  --cpus 0.5 --memory 256m \
  nginx:alpine
```

---

#### **c) Fluentd (Tail Logs → Stdout)**

Use an official Fluentd image:

```bash
docker run -d \
  --name fluentd \
  --network custom-net \
  -v logs-vol:/var/logs \
  -v "$(pwd)"/fluentd/fluent.conf:/fluentd/etc/fluent.conf:ro \
  --restart unless-stopped \
  --cpus 0.5 --memory 256m \
  fluent/fluentd:v1.16-debian
```

---

## **Validation Steps**

### ✅ **Nginx**

```bash
curl http://localhost:8080
# Expected: Hello from Nginx Container!
```

### ✅ **Logs from Cron**

(Wait 1–2 minutes after starting containers)

```bash
docker exec custom-logger tail -n 5 /var/logs/time.log
```

### ✅ **Fluentd Output**

```bash
docker logs fluentd | grep custom.logger
```

### ✅ **Network DNS Resolution**

```bash
docker run --rm --network custom-net alpine:3.20 \
  sh -c 'wget -qO- http://nginx-server | head -n1'
```

---

## **Test Cases**

| Test Case                            | Command                                                                                                       | Expected Result                  |                         |                                        |
| ------------------------------------ | ------------------------------------------------------------------------------------------------------------- | -------------------------------- | ----------------------- | -------------------------------------- |
| 1. Nginx serves HTML                 | `curl http://localhost:8080`                                                                                  | HTML page content loads          |                         |                                        |
| 2. Cron job writes logs              | `docker exec custom-logger tail -n 3 /var/logs/time.log`                                                      | Timestamps appended every minute |                         |                                        |
| 3. Logs persist after restart        | `docker stop custom-logger && docker start custom-logger && docker exec custom-logger cat /var/logs/time.log` | Old logs still exist             |                         |                                        |
| 4. Fluentd prints logs to stdout     | \`docker logs fluentd                                                                                         | grep custom.logger\`             | Log lines visible       |                                        |
| 5. Fluentd still works after restart | \`docker restart fluentd && docker logs fluentd                                                               | grep custom.logger\`             | Log lines visible again |                                        |
| 6. Network communication works       | `docker run --rm --network custom-net alpine:3.20 sh -c 'wget -qO- http://nginx-server'`                      | HTML output returned             |                         |                                        |
| 7. Resource limits applied           | \`docker inspect <container>                                                                                  | grep -A5 HostConfig              | grep NanoCpus\`         | Shows CPU and memory limits configured |

---

## **Clean Restart Test**

To confirm persistence and automatic restart:

```bash
docker stop custom-logger nginx-server fluentd
docker rm custom-logger nginx-server fluentd

# Start again using the same run commands above
```

Check:

* Logs still exist:

```bash
docker run --rm -v logs-vol:/var/logs alpine cat /var/logs/time.log
```

* Cron resumes writing logs.
* Nginx serves HTML.
* Fluentd prints logs.

---

## **Optional: Use Named Volume for HTML**

Instead of a bind-mount:

```bash
docker volume create html-vol
docker run --rm \
  -v html-vol:/dest \
  -v "$(pwd)"/html:/src:ro \
  alpine sh -c 'cp -r /src/* /dest/'

docker run -d \
  --name nginx-server \
  --network custom-net \
  -p 8080:80 \
  -v html-vol:/usr/share/nginx/html:ro \
  --restart unless-stopped \
  --cpus 0.5 --memory 256m \
  nginx:alpine
```

---

## **How Each Requirement is Addressed**

✔ **Custom Logging Container:** Alpine-based, cron runs in foreground, logs timestamps to `/var/logs/time.log` (persisted in `logs-vol`).
✔ **Nginx:** Serves local HTML, port 8080 exposed.
✔ **Fluentd:** Reads logs from shared volume, prints to stdout.
✔ **Docker Networking:** Custom bridge network `custom-net` for name-based resolution.
✔ **Logging Validation:** Logs written and visible via Fluentd output.
✔ **Clean Restart:** Volumes persist logs; restart confirms cron resumes and Nginx works.
✔ **Resource Limits:** `--cpus 0.5` and `--memory 256m` for all containers.

--------
