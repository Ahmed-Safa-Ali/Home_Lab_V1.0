# تحويل لابتوب قديم إلى سيرفر NAS منزلي

مشروع شخصي لتحويل لابتوب HP Pavilion g7 قديم (كان يعمل بنظام Windows 7/10) إلى سيرفر NAS كامل يعمل بنظام Ubuntu Server، مع واجهة إدارة رسومية، مشاركة ملفات عبر الشبكة، وتخزين موسّع بهارد خارجي.

---

## 📋 نظرة عامة على المشروع

| العنصر | التفاصيل |
|---|---|
| **الجهاز** | HP Pavilion g7 Notebook |
| **المعالج** | Intel Core i5-2410M @ 2.30GHz (2 core / 4 thread) |
| **الرام** | 4GB DDR3 (قابل للترقية إلى 6-8GB) |
| **التخزين الداخلي** | SSD 238GB (TEAM T253256GB) |
| **التخزين الخارجي** | HDD 2TB (USB) للملفات والأرشيف |
| **نظام التشغيل** | Ubuntu Server 26.04 LTS |
| **واجهة الإدارة** | CasaOS |
| **مشاركة الملفات** | Samba (SMB) |

---

## 🎯 الهدف من المشروع

- إعادة استخدام هاردوير قديم بدل رميه أو بيعه بسعر رخيص
- بناء NAS شخصي لتخزين ومشاركة الملفات داخل الشبكة المنزلية
- تعلم أساسيات Linux، الشبكات، وإدارة السيرفرات بشكل عملي
- بناء بنية تحتية قابلة للتوسع مستقبلاً (VPN، مانع إعلانات، بث وسائط)

---

## 🛠️ خطوات التنفيذ

### 1. تجهيز الهاردوير
- فحص المواصفات عبر `msinfo32` قبل التنصيب (Windows)
- التأكد من دعم اللوحة الأم لذاكرة DDR3 إضافية (الحد الأقصى الرسمي 8GB)

### 2. إنشاء فلاشة تنصيب Ubuntu Server
- تحميل Ubuntu Server LTS من الموقع الرسمي
- استخدام Rufus لإنشاء USB قابل للإقلاع

### 3. الوصول لإعدادات BIOS/Boot Menu
كانت هذه أول عقبة تقنية:
- زر `F9` لم يستجب بالتوقيت المعتاد أثناء الإقلاع العادي
- **الحل:** استخدام أمر Windows للدخول المباشر لإعدادات الـ Firmware:
  ```
  shutdown /r /fw /t 0
  ```
- داخل BIOS (InsydeH2O)، الفلاشة لم تظهر بقائمة Boot الأولى (Notebook Hard Drive / Internal CD-DVD فقط)
- تم إيجادها داخل **System Configuration → Boot Options → Boot Order** باسم:
  `USB Diskette on Key/USB Hard Disk`
- رفعها لأول القائمة وحفظ الإعدادات (F10)

### 4. تثبيت Ubuntu Server
- خيار Storage: **Use an entire disk** (السيرفر بالكامل، بدون Dual-boot)
- تفعيل LVM دون تشفير LUKS (لتجنب الحاجة لإدخال باسورد فك تشفير عند كل إقلاع في جهاز Headless)

### 5. حل مشكلة الشبكة (أصعب مشكلة بالمشروع)

**الأعراض:**
- الاتصال يظهر `UP` لكن بدون عنوان IPv4، فقط IPv6 محلي
- السجل يظهر تكرار مستمر لـ:
  ```
  eno1: Lost carrier
  eno1: Gained carrier
  ```

**التشخيص:**
```bash
ip a
sudo cat /etc/netplan/*.yaml
sudo networkctl status eno1
sudo journalctl -u systemd-networkd -f   # مراقبة حية للسجل
```

**السبب الجذري:** اتصال فيزيائي غير ثابت بمنفذ الإيثرنت (Loose connection)، وليس خطأً في الإعدادات (ملف netplan كان صحيحاً تماماً: `dhcp4: true`).

**الحل:** تثبيت الكيبل يدوياً بإحكام أكبر في المنفذ حتى استقر الاتصال؛ بعدها حصل الجهاز على IP فوراً:
```
eno1: DHCPv4 address 192.168.0.112/24, gateway 192.168.0.1 acquired
```

### 6. تثبيت واجهة إدارة CasaOS
```bash
curl -fsSL https://get.casaos.io | sudo bash
```
الوصول للواجهة عبر المتصفح: `http://<server-ip>`

### 7. إضافة وتهيئة الهارد الخارجي (2TB)

**المشكلة:** الهارد كان يظهر داخل CasaOS Storage Manager بحالة `NaN undefined` بدل المساحة الفعلية، وزر Format بالواجهة لم يستجب.

**الحل اليدوي عبر Terminal:**
```bash
lsblk                                   # تحديد اسم القرص (sdb2)
sudo umount /media/devmon/2-TB-HDD      # فك الارتباط الحالي
sudo mkfs.ext4 /dev/sdb2                # إعادة التهيئة بصيغة ext4
sudo mkdir -p /mnt/storage
sudo mount /dev/sdb2 /mnt/storage       # إعادة الربط بمسار دائم
df -h /mnt/storage                      # التأكد من نجاح العملية
```

### 8. تثبيت وإعداد Samba لمشاركة الملفات

```bash
sudo apt update
sudo apt install samba -y
sudo smbpasswd -a <username>            # إنشاء باسورد مستخدم خاص بـ Samba
sudo nano /etc/samba/smb.conf
```

إضافة مشاركة جديدة في نهاية الملف:
```ini
[storage]
   path = /mnt/storage
   browseable = yes
   writable = yes
   guest ok = no
   valid users = <username>
```

```bash
sudo systemctl restart smbd
sudo systemctl status smbd
```

**ملاحظة مهمة تم اكتشافها:** يوجد 3 باسوردات منفصلة تماماً في هذا الإعداد ولا يجوز الخلط بينها:
1. باسورد نظام Ubuntu نفسه (تسجيل الدخول + أوامر `sudo`)
2. باسورد Samba (يُستخدم فقط عند الاتصال من Windows عبر `\\server-ip`)
3. باسورد واجهة CasaOS

### 9. تثبيت عنوان IP ثابت (DHCP Reservation)

لضمان عدم تغيّر عنوان السيرفر مستقبلاً، تم ربط MAC Address بعنوان IP ثابت من إعدادات الراوتر مباشرة (TP-Link Archer C50):

```
Advanced → Network → DHCP → Address Reservation → Add New
MAC Address: xx:xx:xx:xx:xx:xx
IP Address: 192.168.0.112
Status: Enabled
```

---

## 🐛 ملخص المشاكل التي تم حلها

| # | المشكلة | الحل |
|---|---|---|
| 1 | عدم استجابة F9 للدخول لـ Boot Menu | استخدام `shutdown /r /fw /t 0` من Windows |
| 2 | الفلاشة غير ظاهرة في قائمة الإقلاع | إيجادها داخل Boot Order باسم "USB Diskette on Key" |
| 3 | فشل DHCP بمحاولة الإقلاع الأولى وقت التنصيب | إعادة المحاولة نجحت تلقائياً |
| 4 | انقطاع متكرر لاتصال الشبكة بعد التنصيب (Lost/Gained Carrier) | تشخيص عبر `journalctl` واكتشاف أن السبب فيزيائي (اتصال كيبل غير ثابت) |
| 5 | الهارد الخارجي يظهر بمساحة `NaN` في واجهة CasaOS | تهيئة يدوية عبر `mkfs.ext4` و`mount` من Terminal |
| 6 | رفض Windows الاتصال بمشاركة الملفات (SMB) | اكتشاف أن Samba لم يكن مثبتاً أصلاً، ثم تثبيته وإعداده |
| 7 | فشل تكرار `sudo smbpasswd` (Authentication failed) | توضيح الفرق بين باسورد النظام وباسورد Samba |
| 8 | تغيّر IP السيرفر مع كل إعادة تشغيل للراوتر | إعداد DHCP Reservation بالراوتر |

---

## 🚀 خطوات مستقبلية مخطط لها

- [ ] تثبيت WireGuard أو Tailscale للوصول الآمن من خارج الشبكة المنزلية
- [ ] تثبيت Pi-hole أو AdGuard Home لحجب الإعلانات على مستوى الشبكة بالكامل
- [ ] تفعيل UFW (Firewall) بإعدادات أمان أساسية
- [ ] تثبيت Jellyfin لتنظيم وبث الوسائط
- [ ] تركيب ذاكرة رام إضافية (DDR3) لرفع القدرة الاستيعابية للخدمات

---

## 💡 الدروس المستفادة

- أغلب مشاكل الشبكات في السيرفرات المنزلية القديمة تكون **فيزيائية** (كيبلات، منافذ) قبل أن تكون في الإعدادات البرمجية
- فصل الباسوردات حسب الخدمة (نظام / Samba / واجهة إدارة) نقطة يجب توثيقها من البداية لتجنب اللخبطة
- التوثيق أثناء التنفيذ (وليس بعده) أهم بكثير من محاولة تذكر الخطوات لاحقاً

---

## 🖥️ المواصفات الكاملة

```
Device: HP Pavilion g7 Notebook PC
CPU: Intel(R) Core(TM) i5-2410M @ 2.30GHz
RAM: 4GB DDR3
Storage: 238GB SSD (internal) + 2TB HDD (external, USB)
Network: Realtek RTL810xE PCI Express Fast Ethernet
OS: Ubuntu Server 26.04.1 LTS
```
