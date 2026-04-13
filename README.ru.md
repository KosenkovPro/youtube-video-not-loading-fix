<p align="right">
  <a href="./README.md">English</a> | <b>Русский</b>
</p>

# 🛠️ Fix: YouTube открывается, но видео не загружается (проблема DNS / CDN на VPS)

## 📖 Обзор

Этот гайд объясняет, как диагностировать и исправить ситуацию, когда:

* YouTube (`youtube.com`) открывается
* Но видео **не загружается или не воспроизводится**

---

## 🌐 Как работает DNS

### 🔄 Стандартный процесс DNS-запроса

```mermaid
sequenceDiagram
    participant Client
    participant LocalResolver as 127.0.0.53 (systemd-resolved)
    participant DNS as Внешний DNS (1.1.1.1 / 8.8.8.8)
    participant Root
    participant TLD as .com
    participant Authoritative as NS google.com

    Client->>LocalResolver: Запрос youtube.com
    LocalResolver->>DNS: Перенаправление запроса
    DNS->>Root: Запрос к root-серверам
    Root-->>DNS: Ответ: зона .com
    DNS->>TLD: Запрос к .com
    TLD-->>DNS: NS для google.com
    DNS->>Authoritative: Запрос google.com
    Authoritative-->>DNS: IP-адрес
    DNS-->>LocalResolver: Ответ
    LocalResolver-->>Client: Ответ
```

---

## 🎥 Как работает YouTube

```mermaid
flowchart LR
    A[Клиент] --> B[youtube.com]
    B --> C[HTML страница]

    A --> D[googlevideo.com]
    D --> E[Видео CDN]

    style A fill:#0d47a1,stroke:#000000,stroke-width:2px,color:#ffffff
    style C fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style B fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style D fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style E fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
```

👉 Важно:

* `youtube.com` → сайт
* `googlevideo.com` → видео

---

## ❌ Сценарий ошибки (твой случай)

```mermaid
flowchart LR
    A[Клиент] --> B[youtube.com OK]
    A --> C[googlevideo.com FAIL]

    B --> D[Страница загружается]
    C --> E[Нет DNS-резолва]

    E --> F[Видео не работает]

    style A fill:#0d47a1,stroke:#000000,stroke-width:2px,color:#ffffff
    style B fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style D fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style C fill:#c62828,stroke:#8e0000,stroke-width:2px,color:#ffffff
    style E fill:#c62828,stroke:#8e0000,stroke-width:2px,color:#ffffff
    style F fill:#c62828,stroke:#8e0000,stroke-width:2px,color:#ffffff
```

---

## 🔍 Причина

Система использовала DNS-серверы провайдера, полученные через DHCP.

Пример:

```text
85.193.xxx.xxx
85.193.xxx.xxx
```

❌ Эти DNS-серверы возвращали некорректные или неполные ответы для CDN-доменов (например, `googlevideo.com`), что нарушало доставку видеоконтента.

---

## ✅ Исправленная схема

```mermaid
flowchart LR
    A[Клиент] --> B[systemd-resolved]
    B --> C[1.1.1.1]
    B --> D[8.8.8.8]

    C --> E[googlevideo.com OK]
    D --> F[youtube.com OK]

    E --> G[Видео работает]

    style A fill:#0d47a1,stroke:#000000,stroke-width:2px,color:#ffffff
    style B fill:#1565c0,stroke:#000000,stroke-width:2px,color:#ffffff

    style C fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style D fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff

    style E fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style F fill:#2e7d32,stroke:#1b5e20,stroke-width:2px,color:#ffffff
    style G fill:#66bb6a,stroke:#1b5e20,stroke-width:2px,color:#ffffff
```

---

## 🌍 Почему DNS даёт разные ответы

```mermaid
flowchart LR
    A[Клиент] --> B[Cloudflare DNS]
    A --> C[Google DNS]

    B --> D[Один оптимальный IP]
    C --> E[Несколько IP]

    D --> F[CDN узел A]
    E --> G[CDN узлы A,B,C]

    style A fill:#0d47a1,stroke:#000000,stroke-width:2px,color:#ffffff

    style B fill:#1565c0,stroke:#0d47a1,stroke-width:2px,color:#ffffff
    style D fill:#1565c0,stroke:#0d47a1,stroke-width:2px,color:#ffffff
    style F fill:#1565c0,stroke:#0d47a1,stroke-width:2px,color:#ffffff

    style C fill:#ef6c00,stroke:#e65100,stroke-width:2px,color:#ffffff
    style E fill:#ef6c00,stroke:#e65100,stroke-width:2px,color:#ffffff
    style G fill:#ef6c00,stroke:#e65100,stroke-width:2px,color:#ffffff
```

---

## 🧠 Ключевая идея

```mermaid
flowchart TD
    A[Сайт работает] --> B[Но контент не грузится]
    B --> C[Проверить CDN домены]
    C --> D[Проверить DNS]
    D --> E[Исправить DNS]
    E --> F[Всё работает]
```

---

# ⚙️ Настройка DNS (что редактировать и что прописать)

🇷🇺 Русский

В зависимости от системы, DNS может управляться через `systemd-resolved`, `netplan` или напрямую через `resolv.conf`.

---

## 🔹 Вариант 1: systemd-resolved (рекомендуется)

Отредактируйте файл:

`sudo nano /etc/systemd/resolved.conf`

Найдите или добавьте строки:

```
[Resolve]
DNS=1.1.1.1 8.8.8.8
FallbackDNS=1.0.0.1 8.8.4.4
DNSSEC=no
DNSOverTLS=no
```

Примените изменения:

`sudo systemctl restart systemd-resolved`

Проверьте:

`resolvectl status`

---

## 🔹 Вариант 2: Netplan (Ubuntu 18.04+)

Откройте конфигурацию (имя файла может отличаться):

`sudo nano /etc/netplan/01-netcfg.yaml`

Добавьте DNS-серверы:

```
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
      nameservers:
        addresses:
          - 1.1.1.1
          - 8.8.8.8
```

Примените:

`sudo netplan apply`

---

## 🔹 Вариант 3: resolv.conf (временное решение)

`sudo nano /etc/resolv.conf`

Добавьте:

```
nameserver 1.1.1.1
nameserver 8.8.8.8
```

⚠️ Важно: этот файл может перезаписываться системой.

---

## 🏁 Вывод

Проблема была вызвана:

❌ DNS провайдера ломал резолв CDN
✔ Исправлено переходом на публичные DNS

После исправления:

* DNS работает корректно
* CDN доступен
* Видео YouTube воспроизводится

---
