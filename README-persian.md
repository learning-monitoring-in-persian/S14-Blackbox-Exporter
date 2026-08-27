[فارسی](README-persian.md) | [English](README.md)

# راه‌اندازی Blackbox Exporter

ابزار **Prometheus Blackbox Exporter** به شما این امکان رو میده تا اندپوینت‌های مختلف رو از طریق پروتکل‌هایی مثل HTTP, HTTPS, DNS, TCP و ICMP بررسی (Probe) کنید.

در حالی که ابزارهایی مثل Node Exporter *داخل* سرور شما رو مانیتور می‌کنن (به این میگن Whitebox Monitoring)، ابزار Blackbox Exporter سیستم شما رو از *بیرون* نگاه می‌کنه (Blackbox). این ابزار به سوالات مهمی جواب میده:
- آیا وب‌سایت من بالاست و کد `200 OK` برمی‌گردونه؟
- چقدر طول می‌کشه تا API من جواب بده (Response Time)؟
- آیا گواهینامه SSL وب‌سایت من قراره به‌زودی منقضی بشه؟
- آیا DNS من داره درست کار می‌کنه؟
- آیا می‌تونم فلان سرور رو پینگ (Ping) کنم؟

---

## نصب Blackbox Exporter به‌صورت سرویس systemd (باینری)

اگر می‌خواید Blackbox Exporter رو مستقیماً روی سرور لینوکسیتون نصب کنید، می‌تونید اون رو به عنوان یک سرویس systemd اجرا کنید.

### ۱. دانلود و نصب فایل باینری

```bash
# نکته‌ مهم
# اگر معماری سیستمی که می‌خواهید روی آن نصب کنید amd64 نیست، دستورات زیر به درستی کار نخواهند کرد.
# برای مثال اگر معماری شما arm64 است باید تمامی `amd64` ها را با `arm64` در کامند‌های زیر جایگزین کنید:

VERSION=$(curl -s https://api.github.com/repos/prometheus/blackbox_exporter/releases/latest | grep '"tag_name"' | cut -d'"' -f4 | sed 's/v//')
wget -O blackbox_exporter.tar.gz https://github.com/prometheus/blackbox_exporter/releases/download/v${VERSION}/blackbox_exporter-${VERSION}.linux-amd64.tar.gz
tar xvfz blackbox_exporter.tar.gz
```

فایل استخراج شده را به مسیر اجرایی سیستم منتقل کنید:
```bash
sudo mv blackbox_exporter-${VERSION}.linux-amd64/blackbox_exporter /usr/local/bin/
rm -rf blackbox_exporter-${VERSION}.linux-amd64 blackbox_exporter.tar.gz
```

برای امنیت بیشتر، یک یوزر سیستمی اختصاصی و یک دایرکتوری برای تنظیمات بسازید:
```bash
sudo useradd --no-create-home --shell /bin/false blackbox_exporter
sudo mkdir -p /etc/blackbox_exporter
```

### ۲. ساخت فایل کانفیگ

فایل `/etc/blackbox_exporter/blackbox.yml` را با محتوای زیر ایجاد کنید:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_http_versions: ["HTTP/1.1", "HTTP/2.0"]
      valid_status_codes: []  # به طور پیش‌فرض کدهای 2xx را درست در نظر می‌گیرد
      method: GET

  icmp_ping:
    prober: icmp
    timeout: 5s
```

> ### نکته
> در فایل کانفیگ بالا ما ماژول‌هایی (مثل `http_2xx` یا `icmp_ping`) تعریف کردیم که مشخص می‌کنن *چطور* باید یک هدف بررسی بشه. نمونه‌های خیلی بیشتری (مثل DNS یا TCP) رو می‌تونید در [مستندات رسمی](https://github.com/prometheus/blackbox_exporter/blob/master/CONFIGURATION.md) پیدا کنید.

دسترسی‌ها را تنظیم کنید:
```bash
sudo chown -R blackbox_exporter:blackbox_exporter /etc/blackbox_exporter
```

### ۳. ساخت فایل سرویس systemd

فایل `/etc/systemd/system/blackbox_exporter.service` را با محتوای زیر ایجاد کنید:
```ini
[Unit]
Description=Prometheus Blackbox Exporter
Wants=network-online.target
After=network-online.target

[Service]
User=blackbox_exporter
Group=blackbox_exporter
Type=simple
ExecStart=/usr/local/bin/blackbox_exporter --config.file=/etc/blackbox_exporter/blackbox.yml

Restart=always

[Install]
WantedBy=multi-user.target
```

در نهایت سرویس را فعال و اجرا کنید:
```bash
sudo systemctl daemon-reload
sudo systemctl enable blackbox_exporter
sudo systemctl start blackbox_exporter
```

---

## راه‌اندازی Blackbox Exporter با Docker Compose (پیشنهادی)

استفاده از داکر برای راه‌اندازی Blackbox Exporter به‌شدت پیشنهاد میشه و خیلی تمیزتره.

ابتدا یک فایل `blackbox.yml` در پوشه خودتون بسازید:
```yaml
modules:
  http_2xx:
    prober: http
    timeout: 5s
    http:
      valid_status_codes: []
      method: GET
```

سپس فایل `docker-compose.yml` را ایجاد کنید:
```yaml
services:
  blackbox-exporter:
    image: prom/blackbox-exporter:latest
    container_name: blackbox_exporter
    restart: unless-stopped
    ports:
      - "9115:9115"
    volumes:
      - ./blackbox.yml:/etc/blackbox_exporter/config.yml:ro
    command:
      - "--config.file=/etc/blackbox_exporter/config.yml"
```

حالا کانتینر را اجرا کنید:
```bash
docker compose up -d
```

---

## نحوه اتصال Prometheus به Blackbox Exporter

> ### نکته مهم
> ابزار Blackbox Exporter با بقیه (مثل Node Exporter) فرق داره! شما نباید فقط خود اکسترپوتر رو اسکریپ کنید. بلکه باید به Prometheus یاد بدید که پارامتر `target` (آدرس وب‌سایت یا IP) و ماژول رو به عنوان کوئری برای Blackbox Exporter بفرسته.

برای این کار باید جاب (Job) زیر رو به فایل `prometheus.yml` خودتون اضافه کنید:

```yaml
scrape_configs:
  - job_name: 'blackbox-http'
    metrics_path: /probe
    params:
      module: [http_2xx]  # می‌خواهیم پاسخ HTTP 200 بگیریم
    static_configs:
      - targets:
        - http://prometheus.io    # سایتی که می‌خواهیم مانیتور کنیم
        - https://google.com      # سایت دومی که می‌خواهیم مانیتور کنیم
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: {IP_ADDRESS}:9115  # آدرس واقعی سروری که Blackbox Exporter روش نصبه
```
*نکته: حتماً `{IP_ADDRESS}` رو با آدرس IP سروری که Blackbox روش نصبه جایگزین کنید. اگر هر دو روی یک سرور اما داخل داکر هستن، باید IP سرور یا اسم کانتینر رو بدید.*

---

## داشبورد گرافانا (Grafana Dashboard)

برای اینکه بتونید وضعیت آپ‌تایم، زمان پینگ‌ها و تاریخ انقضای SSL سایت‌هاتون رو به شکل بصری ببینید، می‌تونید از دشبوردهای آماده کامیونیتی استفاده کنید.
شناسه‌های (ID) `7587` یا `13659` نقطه‌ی شروع خیلی خوبی هستن. فقط کافیه توی گرافانا به بخش Dashboards -> Import برید و این آیدی‌ها رو وارد کنید!
