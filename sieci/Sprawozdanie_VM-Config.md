# Konfiguracja i zabezpieczenie maszyn do pracy w sieci

> Środowisko: VirtualBox, Ubuntu 24.04 LTS + Kali Linux. Celem jest skonfigurowanie komunikacji między maszynami oraz połączenia SSH.

---

## Słownik pojęć

**NAT (Network Address Translation)** — mechanizm tłumaczenia adresów sieciowych. Maszyna wirtualna korzysta z adresu IP hosta, żeby komunikować się z internetem — z zewnątrz wygląda jakby ruch pochodził od hosta, nie od VM. VM nie ma własnego adresu widocznego w sieci lokalnej.

**Mostkowanie (Bridged Networking)** — tryb sieciowy, w którym wirtualna karta sieciowa VM jest "mostkowana" z fizyczną kartą hosta. VM staje się pełnoprawnym uczestnikiem sieci lokalnej i otrzymuje własny adres IP z tej samej puli co inne urządzenia (np. od routera domowego). Z zewnątrz VM wygląda jak osobny komputer w sieci.

**Bridge (most sieciowy)** — urządzenie lub mechanizm łączący dwa segmenty sieci na poziomie warstwy 2 (ramki Ethernet). W kontekście VirtualBox — wirtualny most łączący VM bezpośrednio z fizyczną siecią hosta, pomijając warstwę NAT.

**Gateway (brama sieciowa)** — urządzenie (najczęściej router) stanowiące punkt wyjścia z sieci lokalnej do innych sieci, w tym internetu. Każde urządzenie w sieci ma skonfigurowany adres domyślnej bramy, na który wysyła pakiety przeznaczone poza lokalną sieć. W VirtualBox przy trybie NAT rolę bramy pełni sam hypervisor.

**Współdzielony folder (Shared Folder)** — mechanizm VirtualBox umożliwiający dostęp do folderu na hoście z poziomu systemu gościa (VM). Folder hosta jest montowany w systemie gościa jako wirtualny zasób sieciowy. Wymaga zainstalowanych Guest Additions. Przydatny do wymiany plików między hostem a VM bez użycia sieci czy nośników.

---

## Tryby sieciowe VirtualBox — ściąga

Każda maszyna wirtualna może korzystać z kilku trybów sieciowych. Wybór trybu zależy od tego, co chcemy osiągnąć.

| Tryb | Internet | VM ↔ Host | VM ↔ VM | Kiedy używać |
|---|---|---|---|---|
| **NAT** | ✅ | ❌ | ❌ | VM potrzebuje tylko internetu |
| **NAT Network** | ✅ | ❌ | ✅ | Kilka VM potrzebuje internetu i komunikacji między sobą |
| **Bridged** | ✅ | ✅ | ✅ | VM ma być widoczna w sieci LAN jak osobny komputer |
| **Host-only** | ❌ | ✅ | ✅ | Izolowane laboratorium, VM ↔ host bez internetu |
| **Internal** | ❌ | ❌ | ✅ | Pełna izolacja, tylko VM między sobą |

### Wybór dla tego laboratorium

Do komunikacji między Ubuntu a Kali przy zachowaniu dostępu do internetu używamy **dwóch kart sieciowych** na każdej maszynie:

- **Karta 1 (NAT)** — dostęp do internetu
- **Karta 2 (Host-only)** — komunikacja między maszynami

---

## Konfiguracja sieci

### Tworzenie sieci Host-only

W VirtualBox przed uruchomieniem maszyn:

```
Plik → Menedżer sieci Host-only → Utwórz (ikona +)
```

Domyślnie tworzy się sieć `vboxnet0` z pulą adresów `192.168.56.0/24`. Można zostawić domyślne ustawienia.

### Dodanie karty Host-only do maszyn

Dla każdej VM (Ubuntu i Kali) przy wyłączonej maszynie:

```
Ustawienia → Sieć → Karta 2
→ Włącz kartę sieciową
→ Podłączona do: Host-only Adapter
→ Nazwa: vboxnet0
```

### Sprawdzenie adresów IP

Po uruchomieniu obu maszyn:

```bash
ip a
```

Karta host-only (zazwyczaj `eth1` lub `enp0s8`) powinna mieć adres z puli `192.168.56.x`. Jeśli adres nie został przydzielony automatycznie:

```bash
sudo dhclient enp0s8
```

Przykładowe adresy po konfiguracji:
- Ubuntu: `192.168.56.100`
- Kali: `192.168.56.101`

---

## Weryfikacja połączenia — ping

Z Ubuntu do Kali:

```bash
ping 192.168.56.101
```

Z Kali do Ubuntu:

```bash
ping 192.168.56.100
```

Oczekiwany wynik:
```
PING 192.168.56.101 (192.168.56.101) 56(84) bytes of data.
64 bytes from 192.168.56.101: icmp_seq=1 ttl=64 time=0.45 ms
```

---

## Konfiguracja zapory sieciowej (UFW)

Na Ubuntu domyślnie działa UFW. Jeśli `ping` nie przechodzi, należy zezwolić na ruch z sieci host-only.

### Sprawdzenie stanu UFW

```bash
sudo ufw status verbose
```

### Zezwolenie na ruch z sieci laboratoryjnej

```bash
sudo ufw allow from 192.168.56.0/24
sudo ufw reload
```

### Weryfikacja reguł

```bash
sudo ufw status numbered
```

Na Kali domyślnie nie ma aktywnego firewalla — jeśli ping z Kali do Ubuntu nie działa, problem leży po stronie Ubuntu.

---

## Konfiguracja SSH

### Instalacja i uruchomienie serwera SSH

**Ubuntu** — zazwyczaj SSH nie jest zainstalowany domyślnie:

```bash
sudo apt update
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

**Kali** — SSH jest zainstalowany, ale domyślnie wyłączony:

```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Sprawdzenie czy SSH nasłuchuje

```bash
ss -tlnp | grep 22
```

Oczekiwany wynik: `LISTEN 0 128 0.0.0.0:22`

### Połączenie SSH między maszynami

Z Ubuntu do Kali:

```bash
ssh kali@192.168.56.101
```

Z Kali do Ubuntu (podmień `user` na nazwę użytkownika):

```bash
ssh user@192.168.56.100
```

Przy pierwszym połączeniu SSH pyta o akceptację klucza hosta — wpisz `yes`.

### Uwierzytelnianie kluczem (bez hasła)

Bezpieczniejsza i wygodniejsza alternatywa dla hasła:

```bash
# Generowanie pary kluczy (na maszynie źródłowej)
ssh-keygen -t ed25519 -C "lab-key"
# Klucze zostaną zapisane w ~/.ssh/id_ed25519 i ~/.ssh/id_ed25519.pub

# Skopiowanie klucza publicznego na maszynę docelową
ssh-copy-id user@192.168.56.100
```

Od tej pory logowanie odbywa się bez podawania hasła.

### Podstawowe zabezpieczenie SSH

Edycja `/etc/ssh/sshd_config`:

```bash
sudo nano /etc/ssh/sshd_config
```

Zalecane opcje dla środowiska laboratoryjnego:

```
PermitRootLogin no          # wyłącz logowanie jako root
PasswordAuthentication no   # wyłącz logowanie hasłem (po skonfigurowaniu kluczy)
MaxAuthTries 3              # maksymalnie 3 próby logowania
```

Po zmianach restart SSH:

```bash
sudo systemctl restart ssh
```

---

## Podsumowanie konfiguracji

| Element | Ubuntu | Kali |
|---|---|---|
| Karta 1 | NAT | NAT |
| Karta 2 | Host-only (192.168.56.100) | Host-only (192.168.56.101) |
| SSH serwer | zainstalowany, aktywny | zainstalowany, aktywny |
| Firewall | UFW, reguła dla 192.168.56.0/24 | brak (domyślnie) |
| Ping VM↔VM | ✅ | ✅ |
| SSH VM↔VM | ✅ | ✅ |