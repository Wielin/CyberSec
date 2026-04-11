# Sprawozdanie — VirtualBox, Ubuntu 24.04 & Kali Linux

> Poradnik dla osoby z podstawową wiedzą o wirtualizacji. Zakładam, że wiesz czym jest VM i masz zainstalowany VirtualBox.

---

## 1. Instalacja maszyn wirtualnych

### Ubuntu 24.04 LTS

Podczas tworzenia nowej maszyny w VirtualBox wystarczy wskazać ISO — kreator automatycznie wykryje system i zaproponuje **Unattended Install**, który sam przejdzie przez instalację bez klikania. Warto jednak zrobić to ręcznie, żeby mieć kontrolę nad partycjonowaniem i nazwą użytkownika.

Zalecane parametry dla Ubuntu:

| Parametr | Wartość |
|---|---|
| RAM | 2048–4096 MB |
| CPU | 2 rdzenie |
| Dysk | 25 GB (dynamicznie alokowany) |
| Typ dysku | VDI |

Podczas instalacji Ubuntu wybierz **„Minimalna instalacja"** — mniej śmieci, szybsze VM.

### Kali Linux

Kali najlepiej pobrać jako gotowy obraz VirtualBox (`.ova`) z oficjalnej strony `kali.org/get-kali` — sekcja **Virtual Machines**. Import przez `Plik → Importuj urządzenie wirtualne`. Oszczędza to instalację od zera i daje gotowy, skonfigurowany system.

Domyślne dane logowania po imporcie: `kali` / `kali`.

---

## 2. Konfiguracja opcji maszyny wirtualnej

### Guest Additions (podstawa wszystkiego)

Bez Guest Additions nie działa schowek, integracja myszy ani współdzielone foldery. Instalacja w działającym Ubuntu:

```bash
sudo apt update
sudo apt install virtualbox-guest-additions-iso
# Następnie w menu VM: Urządzenia → Wstaw obraz CD z dodatkami
sudo mount /dev/cdrom /mnt
sudo /mnt/VBoxLinuxAdditions.run
sudo reboot
```

Na Kali Guest Additions są zazwyczaj już zainstalowane w obrazie `.ova`.

### Schowek i przeciągnij-upuść

`Ustawienia → Ogólne → Zaawansowane`:
- **Schowek:** Dwukierunkowy
- **Przeciągnij i upuść:** Dwukierunkowy

Zmiany działają bez restartu VM.

### Współdzielony folder

`Ustawienia → Współdzielone foldery → [+]`:
- Wskaż folder na hoście
- Zaznacz **„Automatyczne montowanie"** i **„Stały"**
- Punkt montowania np. `/mnt/shared`

W Ubuntu folder pojawi się w `/media/sf_NazwaFolderu`. Jeśli brak dostępu:

```bash
sudo usermod -aG vboxsf $USER
# wyloguj i zaloguj ponownie
```

### Wyświetlanie

`Ustawienia → Ekran`:
- **Pamięć wideo:** minimum 64 MB, dla wygody 128 MB
- **Akceleracja 3D:** można włączyć, ale bywa niestabilne — jeśli VM się sypie, wyłącz
- W działającej VM: `Widok → Dopasuj ekran gościa` lub `Widok → Pełny ekran` (skrót: `Host + F`, gdzie Host to prawy Ctrl)

---

## 3. Migawki (snapshots)

Migawka to zamrożony stan VM w danym momencie — dysk, RAM, ustawienia. Pozwala wrócić do poprzedniego stanu w kilka sekund.

### Tworzenie migawki bazowej

Po czystej instalacji, przed jakimikolwiek zmianami:

```
Menu VM → Maszyna → Zrób migawkę
```

Nazwa: np. `Ubuntu_base` lub `Ubuntu_24-base`. Opis: `Czysty system po instalacji, bez konfiguracji`.

Warto zrobić migawkę **przy wyłączonej VM** — jest szybsza i mniejsza (nie zapisuje stanu RAM).

### Eksperyment z migawkami

Aby pokazać działanie migawek:

1. Zrób migawkę `base` — czysty system
2. Zmień tapetę pulpitu i utwórz plik na pulpicie:
   ```bash
   echo "test migawki" > ~/Desktop/test.txt
   ```
3. Zrób drugą migawkę `po_zmianach`
4. Przywróć migawkę `base` — plik i tapeta znikają

### Przywracanie migawki

```
Menedżer migawek (ikona aparatu w prawym górnym rogu listy VM)
→ Wybierz migawkę → Przywróć
```

Aktualny stan można zapisać jako nową migawkę przed przywróceniem — VirtualBox zapyta o to automatycznie.

---

## 4. Konfiguracja sieciowa — ściąga

| Tryb | Dostęp do internetu | VM widzi hosta | Host widzi VM | VM widzi VM |
|---|---|---|---|---|
| **NAT** | ✅ | ❌ | ❌ | ❌ |
| **NAT Network** | ✅ | ❌ | ❌ | ✅ |
| **Bridged** | ✅ | ✅ | ✅ | ✅ |
| **Host-only** | ❌ | ✅ | ✅ | ✅ |
| **Internal** | ❌ | ❌ | ❌ | ✅ |

### Kiedy używać którego trybu

- **NAT** — domyślny, VM potrzebuje internetu, nic więcej. Najprostszy, bezpieczny.
- **NAT Network** — kilka VM potrzebuje internetu i musi się między sobą widzieć.
- **Bridged** — VM ma być widoczna w sieci lokalnej jak osobny komputer (np. serwer). Używa fizycznej karty hosta.
- **Host-only** — izolowane laboratorium: VM ↔ host, bez internetu. Idealne do testów.
- **Internal** — całkowita izolacja, tylko VM między sobą. Przydatne przy symulowaniu sieci.

### Konfiguracja dla komunikacji Ubuntu ↔ Kali

Obie maszyny muszą być w tej samej sieci. Najwygodniej: **Host-only adapter** jako druga karta sieciowa (pierwsza NAT dla internetu).

W VirtualBox przed uruchomieniem VM:
```
Plik → Menedżer sieci Host-only → Utwórz
```
Domyślna sieć: `192.168.56.0/24`, brama: `192.168.56.1`.

Dla każdej VM `Ustawienia → Sieć`:
- Karta 1: NAT (internet)
- Karta 2: Host-only Adapter → `vboxnet0`

Po uruchomieniu sprawdź adresy IP:
```bash
ip a
```

Karta host-only będzie miała adres z puli `192.168.56.x`.

### Firewall (UFW na Ubuntu)

Jeśli `ping` nie działa, sprawdź UFW:

```bash
sudo ufw status
sudo ufw allow from 192.168.56.0/24
sudo ufw reload
```

Na Kali domyślnie nie ma aktywnego firewalla — jeśli Ubuntu nie odpowiada na ping, problem leży po stronie Ubuntu.

Weryfikacja połączenia:
```bash
# Z Ubuntu pinguj Kali (i odwrotnie)
ping 192.168.56.101
```

---

## 5. Konfiguracja SSH między maszynami

### Instalacja serwera SSH

Na Ubuntu (jeśli nie ma):
```bash
sudo apt install openssh-server
sudo systemctl enable ssh
sudo systemctl start ssh
```

Na Kali SSH jest domyślnie zainstalowany, ale wyłączony:
```bash
sudo systemctl enable ssh
sudo systemctl start ssh
```

### Połączenie

```bash
# Z Ubuntu do Kali
ssh kali@192.168.56.101

# Z Kali do Ubuntu
ssh user@192.168.56.100
```

### Klucze SSH (opcjonalnie, ale wygodnie)

Zamiast hasła przy każdym połączeniu:

```bash
# Na maszynie źródłowej
ssh-keygen -t ed25519

# Skopiuj klucz publiczny na docelową maszynę
ssh-copy-id user@192.168.56.100
```

Od tej pory logowanie bez hasła.

### Weryfikacja

```bash
# Sprawdź czy port 22 nasłuchuje
ss -tlnp | grep 22

# Test połączenia
ssh -v user@192.168.56.100
```

---

## Podsumowanie — kolejność działań

1. Instalacja VM (Ubuntu z ISO, Kali z `.ova`)
2. Instalacja Guest Additions → schowek, foldery, ekran
3. Migawka bazowa przy wyłączonej VM
4. Konfiguracja sieci: NAT + Host-only na obu maszynach
5. Weryfikacja `ping` między maszynami
6. Konfiguracja i test SSH