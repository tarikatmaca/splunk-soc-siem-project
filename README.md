# 🛡️ Splunk SIEM ile Güvenlik İzleme Projesi

Windows Event Log, Sysmon ve Linux authentication loglarının Splunk Enterprise'a aktarılması; RDP brute force, komut satırı keşif faaliyeti ve ağ taraması gibi saldırı senaryolarının SPL sorguları ile tespit edilip MITRE ATT&CK Framework'e eşlenerek dashboard üzerinde görselleştirilmesi.

![Security Monitoring Dashboard](screenshots/59-final-dashboard-HERO.png)

> 📄 Bu README, projenin öne çıkan adımlarını özetler. Kurulumun **her adımına ait 59 ekran görüntüsünü** içeren tam, 65 sayfalık adım-adım raporu [`Splunk-SIEM-Proje-Raporu.pdf`](./Splunk-SIEM-Proje-Raporu.pdf) dosyasında bulabilirsiniz.

---

## 🎯 Proje Özeti

Bu proje kapsamında Splunk Enterprise üzerinde uçtan uca bir SIEM (Security Information and Event Management) altyapısı kuruldu. Windows Event Logları, Sysmon telemetrisi ve Linux authentication logları Splunk Universal Forwarder ile merkezi bir Splunk sunucusuna aktarıldı. Ardından üç farklı saldırı senaryosu simüle edilerek özel SPL sorguları ile tespit edildi ve MITRE ATT&CK Framework'e eşlenerek interaktif bir dashboard üzerinde görselleştirildi.

## 🏗️ Mimari

```
Linux Saldırgan Makinesi (Hydra / Nmap)
        │
        ▼
Windows Uç Noktası
  ├─ Windows Event Logs
  ├─ Sysmon Operational Logs (topluluk config: sysmon-modular)
  └─ Splunk Universal Forwarder
        │  (log aktarımı)
        ▼
Splunk Enterprise Sunucusu
  ├─ İndeksleme
  ├─ SPL ile Arama/Analiz
  └─ Dashboard & Report'lar
```

## ⚙️ Kullanılan Teknolojiler

| Teknoloji | Amaç |
|---|---|
| Splunk Enterprise | SIEM platformu |
| Splunk Universal Forwarder 10.4.1 | Log toplama (Linux & Windows) |
| Microsoft Sysmon v15.21 | Uç nokta (endpoint) izleme |
| Olaf Hartong `sysmon-modular` | Topluluk Sysmon yapılandırma dosyası |
| Windows Event Log | Kimlik doğrulama logları |
| Linux `auth.log` | Linux kimlik doğrulama izleme |
| Hydra | RDP brute force saldırı simülasyonu |
| Nmap / Zenmap | Ağ keşfi (reconnaissance) |
| MITRE ATT&CK | Tehdit sınıflandırma |
| SPL | Tespit sorguları |

---

## 1️⃣ Log Kaynaklarının Splunk'a Bağlanması

### Linux Universal Forwarder Kurulumu

<img src="screenshots/01-linux-ssh-baglanti.png" width="700"/>

Linux sunucusuna SSH ile bağlanılarak Splunk Universal Forwarder paketi resmi Splunk deposundan indirildi ve kuruldu.

<img src="screenshots/05-linux-kurulum-basarili.png" width="700"/>

Sanal makine ortamına özgü bir CPU uyumluluk kontrolü aşılarak kurulum başarıyla tamamlandı, forward server (`10.10.200.3:9997`) tanımlandı ve `/var/log/auth.log` izlenen kaynak olarak eklendi.

<img src="screenshots/16-linux-loglar-splunkta.png" width="700"/>

`/var/log/auth.log` kaynağından gelen olayların Splunk'ta gerçek zamanlı olarak indekslendiği doğrulandı (25.954 olay).

### Windows Universal Forwarder Kurulumu

<img src="screenshots/17-windows-rdp-baglanti.png" width="700"/>

Windows uç noktasına RDP ile erişilerek kurulum sihirbazı çalıştırıldı; Windows Event Log kaynakları (Application, Security, System, ForwardedEvents, Setup) seçilerek receiving indexer (`10.10.200.3:9997`) tanımlandı.

<img src="screenshots/30-windows-kurulum-tamamlandi.png" width="700"/>

Splunk Universal Forwarder kurulumu başarıyla tamamlandı ve servis otomatik olarak başlatıldı.

<img src="screenshots/31-windows-loglar-splunkta.png" width="700"/>

Windows Security Event Log kayıtlarının Splunk'a gerçek zamanlı ulaştığı doğrulandı.

### Sysmon Kurulumu ve Yapılandırılması

<img src="screenshots/32-sysmon-indirme-sayfasi.png" width="700"/>

Sysmon, Microsoft Sysinternals'ın resmi sayfasından indirildi.

<img src="screenshots/44-sysmon-yapilandirma-komutu.png" width="700"/>

Sysmon, Olaf Hartong'un topluluk yapılandırma dosyası kullanılarak yönetici komut isteminde yapılandırıldı:

```
sysmon -c sysmonconfig-with-filedelete.xml
```

<img src="screenshots/49-sysmon-inputsconf-eklendi.png" width="700"/>

Universal Forwarder'ın `inputs.conf` dosyasına Sysmon Operational kanalı eklenerek loglar Splunk'a aktarılmaya başlandı:

```ini
[WinEventLog://Microsoft-Windows-Sysmon/Operational]
checkpointInterval = 5
current_only = 0
disabled = 0
start_from = oldest
```

<img src="screenshots/52-sysmon-loglar-splunkta.png" width="700"/>

Sysmon Operational loglarının Splunk'a başarıyla ulaştığı, kaynak (source) istatistikleri üzerinden doğrulandı.

> 📸 Windows kurulum sihirbazının tüm ekranları, Sysmon'un indirilip yapılandırılmasının her adımı (extract, GitHub'dan config indirme, dosya taşıma vb.) ve Linux tarafındaki tüm doğrulama komutları PDF raporunda eksiksiz belgelenmiştir.

---

## 2️⃣ Use Case 1 — RDP Brute Force Tespiti

<img src="screenshots/53-hydra-bruteforce-saldiri.png" width="700"/>

Saldırı senaryosunu üretmek amacıyla Linux makinesinden Hydra kullanılarak Windows RDP servisine karşı sözlük tabanlı bir brute force saldırısı gerçekleştirildi.

<img src="screenshots/54-bruteforce-spl-sorgu-sonuc.png" width="700"/>

Windows Security Event ID 4624/4625 kayıtları kaynak IP ve kullanıcı adına göre gruplanarak 5 ve üzeri başarısız girişi olan oturumlar tespit edildi. Sonuç: **5.555 başarısız, 1 başarılı** giriş — klasik bir başarılı brute force saldırısı paterni.

```spl
(EventCode=4624 OR EventCode=4625) earliest=-24h
Source_Network_Address="10.10.200.51"
Account_Name="std"
| stats count(eval(EventCode=4625)) as Failed
        count(eval(EventCode=4624)) as Success
        earliest(_time) as FirstAttempt
        latest(_time) as LastEvent
        values(Failure_Reason) as FailureReason
        values(Logon_Type) as LogonType
  by Source_Network_Address ComputerName
| where Failed>=5
| sort -Failed
```

**MITRE ATT&CK:** T1110 — Brute Force

---

## 3️⃣ Use Case 2 — Whoami ile Kullanıcı Keşfi Tespiti

<img src="screenshots/56-whoami-spl-sorgu-sonuc.png" width="700"/>

Ele geçirilen hesap üzerinden çalıştırılan `whoami.exe`, Sysmon Event ID 1 (Process Creation) kayıtları üzerinden tespit edildi.

```spl
source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
EventCode=1
Image="*whoami.exe"
| eval Technique="Discovery (MITRE T1033)"
| table Time User Computer Image ParentImage CommandLine Technique
| sort -Time
```

**MITRE ATT&CK:** T1033 — System Owner/User Discovery

---

## 4️⃣ Use Case 3 — Nmap ile Ağ Taraması Tespiti

<img src="screenshots/57-nmap-zenmap-tarama.png" width="700"/>

Saldırgan, ele geçirdiği Windows makinesi üzerinden Zenmap (Nmap GUI) kullanarak ağ taraması gerçekleştirdi.

```spl
source="WinEventLog:Microsoft-Windows-Sysmon/Operational"
EventCode=1
(Image="*nmap.exe" OR Image="*zenmap.exe")
| eval Technique="Discovery (MITRE T1046)"
| table Time User Computer Image ParentImage CommandLine Technique
| sort -Time
```

**MITRE ATT&CK:** T1046 — Network Service Discovery

---

## 5️⃣ Nihai Dashboard

Üç use case tek bir **Security Monitoring Dashboard** üzerinde birleştirildi:

- **Failed Logins:** 5.560
- **Successful Logins:** 1
- **Whoami Execution:** 3
- **Network Scan Detection:** 1
- Brute Force saldırı zaman çizelgesi
- Her use case için detaylı olay tablosu

## 🧩 MITRE ATT&CK Eşlemesi

| Saldırı Aşaması | Teknik | ATT&CK ID |
|---|---|---|
| Initial Access | Brute Force | T1110 |
| Discovery | System Owner/User Discovery | T1033 |
| Discovery | Network Service Discovery | T1046 |

## ✅ Gereksinim Karşılama Durumu

| # | Gereksinim | Durum |
|---|---|---|
| G1 | VM ve Splunk sunucusuna erişim | ✅ |
| G2 | Windows'a Sysmon kurulumu | ✅ |
| G3 | Windows Event Log aktarımı | ✅ |
| G4 | Sysmon loglarının aktarımı | ✅ |
| G5 | Linux auth.log aktarımı | ✅ |
| G6 | Gerçek zamanlı log doğrulaması | ✅ |
| G7 | Dashboard oluşturulması | ✅ |
| G8 | En az 3 use case | ✅ |
| G9 | Use case'lerin dashboard'da gösterimi | ✅ |
| G10 | SPL sorguları, Report, Dashboard | ✅ |

Detaylı gereksinim–ekran görüntüsü eşlemesi için PDF raporunun **Bölüm 5 — Gereksinim Kontrolü** kısmına bakınız.

## ✅ Sonuç

Proje, Windows ve Linux ortamlarından toplanan logların merkezi bir SIEM platformunda nasıl anlamlı güvenlik içgörülerine dönüştürülebileceğini; kurulumun her adımından (Universal Forwarder, Sysmon) tespit sorgu geliştirmeye ve dashboard tasarımına kadar uçtan uca göstermektedir. Üç use case gerçekçi bir saldırı zincirini (kimlik doğrulama saldırısı → keşif → ağ taraması) temsil etmekte ve MITRE ATT&CK Framework ile eşlenerek SOC analist bakış açısıyla raporlanmaktadır.

---

## 📁 Repo Yapısı

```
splunk-siem-project/
├── README.md
├── Splunk-SIEM-Proje-Raporu.pdf      # 65 sayfalık tam adım-adım rapor (59 ekran görüntüsü)
└── screenshots/                       # Tüm ekran görüntüleri (ham dosyalar)
    ├── 01-linux-ssh-baglanti.png
    ├── ...
    └── 59-final-dashboard-HERO.png
```

---

<p align="center">
<sub>Splunk · SIEM · SOC · Sysmon · MITRE ATT&CK · Blue Team</sub>
</p>
