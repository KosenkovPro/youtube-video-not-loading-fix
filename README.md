<p align="right">
  <b>English</b> | <a href="./README.ru.md">Русский</a>
</p>

# 🛠️ Fix: YouTube opens but videos don't load (DNS / CDN issue on VPS)

## 📖 Overview

This guide explains how to diagnose and fix a situation where:

* You can open YouTube (`youtube.com`)
* But videos do **not load or play**

---

## 🌐 How DNS resolution works

### 🔄 Standard DNS flow

```mermaid
sequenceDiagram
    participant Client
    participant LocalResolver as 127.0.0.53 (systemd-resolved)
    participant DNS as External DNS (1.1.1.1 / 8.8.8.8)
    participant Root
    participant TLD as .com
    participant Authoritative as google.com NS

    Client->>LocalResolver: Query youtube.com
    LocalResolver->>DNS: Forward request
    DNS->>Root: Ask root servers
    Root-->>DNS: Refer to .com
    DNS->>TLD: Ask .com servers
    TLD-->>DNS: Refer to google NS
    DNS->>Authoritative: Ask google.com
    Authoritative-->>DNS: Return IP
    DNS-->>LocalResolver: Return IP
    LocalResolver-->>Client: Return IP
```

---

## 🎥 How YouTube actually works

```mermaid
flowchart LR
    A[Client] --> B[youtube.com]
    B --> C[HTML Page]

    A --> D[googlevideo.com]
    D --> E[Video Stream CDN]

    style B fill:#90caf9
    style D fill:#ffcc80
```

👉 Important:

* `youtube.com` → loads website
* `googlevideo.com` → streams video

---

## ❌ Failure scenario (your case)

```mermaid
flowchart LR
    A[Client] --> B[youtube.com OK]
    A --> C[googlevideo.com FAIL]

    B --> D[HTML loads]
    C --> E[No DNS resolution]

    E --> F[Video does not load]

    style B fill:#a5d6a7
    style C fill:#ef9a9a
    style F fill:#ef5350
```

---

## 🔍 Root Cause

Provider DNS servers assigned via DHCP

Example:
```
85.193.xxx.xxx
85.193.xxx.xxx
```

❌ These DNS servers returned incomplete or incorrect responses for CDN domains such as `googlevideo.com`, breaking video delivery.

---

## ✅ Fixed architecture

```mermaid
flowchart LR
    A[Client] --> B[systemd-resolved]
    B --> C[1.1.1.1]
    B --> D[8.8.8.8]

    C --> E[googlevideo.com OK]
    D --> F[youtube.com OK]

    E --> G[Video works]
```

---

## 🌍 Why DNS responses differ

```mermaid
flowchart LR
    A[Client] --> B[Cloudflare DNS]
    A --> C[Google DNS]

    B --> D[Single optimized IP]
    C --> E[Multiple IPs]

    D --> F[CDN Node A]
    E --> G[CDN Nodes A,B,C]

    style B fill:#81d4fa
    style C fill:#ffcc80
```

---

## 🧠 Key Insight

```mermaid
flowchart TD
    A[Site works] --> B[But content fails]
    B --> C[Check CDN domains]
    C --> D[Check DNS]
    D --> E[Fix DNS]
    E --> F[Everything works]
```

---

## 🏁 Conclusion

The issue was caused by:

❌ Broken DNS from hosting provider
✔ Fixed by switching to public DNS

After fix:

* DNS resolution works
* CDN accessible
* YouTube video playback restored

---
