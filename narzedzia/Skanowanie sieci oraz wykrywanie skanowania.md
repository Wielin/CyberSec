# Sprawozdanie ze skanowania i wykrywania skanowania sieci komputerowych

## 1. Rozeznanie oraz skanowanie sieci

### 1.1 Narzędzia do skanowania

#### NMAP

Nmap (Network Mapper) to open-source'owe narzędzie służące do skanowania sieci i audytu bezpieczeństwa. Umożliwia wykrywanie aktywnych hostów w sieci, identyfikację otwartych portów, rozpoznawanie usług oraz ich wersji, a także określanie systemu operacyjnego zdalnych urządzeń. Jest to podstawowe narzędzie w arsenale każdego administratora i specjalisty ds. bezpieczeństwa.

#### ZENMAP

Zenmap to oficjalny graficzny interfejs użytkownika (GUI) dla nmap. Zapewnia przyjazną formę obsługi nmap poprzez graficzne menu, umożliwia wizualizację topologii sieci oraz zapisywanie i porównywanie wyników skanowania. Szczególnie przydatny dla użytkowników preferujących pracę w środowisku graficznym.

#### Kiedy używać nmap a kiedy zenmap

Nmap jest idealny do automatyzacji, skryptów i pracy w środowisku serwerowym (CLI). Zenmap sprawdza się podczas nauki, analizy wizualnej sieci oraz przy porównywaniu wyników skanowania. Dla bardziej zaawansowanych opcji warto skorzystać z `nmap -h` lub `man nmap`.

Przykładem zaawansowanej techniki jest użycie przełącznika `-D` (decoy), który pozwala na podszywanie się pod inne adresy IP podczas skanowania: `nmap -D RND:10 192.168.1.10` generuje 10 losowych adresów-pułapek, co znacznie utrudnia administratorowi określenie, który adres IP faktycznie wykonuje skanowanie.

### 1.2 Podstawowe skany i ich zastosowanie

**Skan 1: TCP SYN Scan (`nmap -sS 192.168.1.0/24`)**

- Najczęściej stosowany, szybki i stosunkowo dyskretny
- Nie nawiązuje pełnego połączenia TCP
- Wymaga uprawnień root

**Skan 2: TCP Connect Scan (`nmap -sT 192.168.1.10`)**

- Nie wymaga uprawnień root
- Bardziej wykrywalny, nawiązuje pełne połączenie
- Przydatny gdy nie mamy dostępu root

**Skan 3: UDP Scan (`nmap -sU 192.168.1.10`)**

- Skanowanie portów UDP
- Wolniejszy niż TCP
- Ważny dla wykrywania usług jak DNS, SNMP, DHCP

**Skan 4: Version Detection (`nmap -sV 192.168.1.10`)**

- Wykrywa wersje usług działających na otwartych portach
- Pomocny przy identyfikacji potencjalnych luk
- Bardziej inwazyjny i czasochłonny

## 2. Ochrona oraz wykrywanie skanowania

### 2.1 Narzędzia do wykrywania

#### tcpdump/Wireshark

Tcpdump to konsolowe narzędzie do przechwytywania i analizy pakietów sieciowych w czasie rzeczywistym. Wireshark oferuje graficzny interfejs z zaawansowanymi możliwościami filtrowania i analizy ruchu. Oba narzędzia są niezbędne do monitorowania aktywności sieciowej i wykrywania anomalii.

#### Systemy IDS/IPS

Ciekawostką w kontekście wykrywania skanowania są systemy IDS (Intrusion Detection System) oraz IPS (Intrusion Prevention System). Są to wyspecjalizowane rozwiązania, które w czasie rzeczywistym analizują ruch sieciowy, porównując go z bazą znanych wzorców ataków. Przykładem takiego systemu jest SNORT - jeden z najpopularniejszych open-source'owych IDS/IPS, który potrafi automatycznie wykrywać skanowanie portów, próby exploitów oraz inne podejrzane aktywności sieciowe na podstawie predefiniowanych reguł.

### 2.2 Jak odróżnić ruch sieciowy od skanowania

#### Wnioski z tcpdump/Wireshark

Normalny ruch sieciowy charakteryzuje się:

- Nawiązywaniem pełnych połączeń TCP (SYN → SYN-ACK → ACK)
- Wymianą danych aplikacyjnych
- Logicznym przepływem między znanymi hostami

Skanowanie wyróżnia się:

- Licznymi próbami połączeń do wielu portów w krótkim czasie
- Nieukończonymi handshake'ami TCP (np. tylko pakiety SYN)
- Sekwencyjnym lub losowym skanowaniem zakresów portów
- Połączeniami do typowo nieużywanych portów

#### Logi systemowe

Logi systemowe (np. `/var/log/auth.log`, `/var/log/syslog`) rejestrują nieautoryzowane próby połączeń i mogą wskazywać na skanowanie. Warto monitorować nagłe wzrosty liczby odrzuconych połączeń z jednego źródła.

## 3. Odpowiedzi na kluczowe pytania

### Jak rozpoznać skanowanie?

**Różnice między skanem SYN a normalnym ruchem:**

Skan SYN wysyła tylko pakiety SYN bez dokończenia trójstronnego uścisku dłoni (three-way handshake). W normalnym ruchu po otrzymaniu SYN-ACK następuje pakiet ACK i wymiana danych. Przy skanowaniu obserwujemy:

- Wiele pakietów SYN do różnych portów
- Brak pakietów ACK lub natychmiastowe RST po SYN-ACK
- Równe odstępy czasowe między próbami (automatyzacja)

**Ograniczenia wykrywania:**

- Wolne skanowanie (slow scan) może być trudne do odróżnienia od normalnego ruchu
- Skanowanie rozproszone z wielu źródeł (distributed scan) utrudnia identyfikację
- Szyfrowany ruch (VPN, TLS) ogranicza możliwości inspekcji
- Duża ilość legalnego ruchu może maskować skanowanie
- Techniki jak decoy (`-D`) generują fałszywe źródła ataków, co komplikuje identyfikację rzeczywistego skanera

## 4. Wnioski

### Jak wykrywać skanowanie?

Skuteczne wykrywanie skanowania wymaga:

- Ciągłego monitorowania ruchu sieciowego (tcpdump, Wireshark, opcjonalnie IDS/IPS)
- Analizy logów systemowych pod kątem anomalii
- Wykrywania wzorców: wielu połączeń z jednego źródła, skanowania sekwencyjnego portów
- Korelacji zdarzeń z różnych źródeł (firewall, logi aplikacji)
- Ustawiania alertów na podejrzane aktywności

### Czy administrator musi znać narzędzia używane przez atakującego?

Administrator nie musi znać konkretnych narzędzi atakującego, ale powinien rozumieć techniki i wzorce ataków. Skanowanie niezależnie od narzędzia (nmap, masscan, zmap) pozostawia charakterystyczne ślady w ruchu sieciowym. Kluczowa jest wiedza o:

- Sposobach działania różnych typów skanów (SYN, Connect, UDP, FIN)
- Wzorcach anomalii w ruchu sieciowym
- Metodach obejścia zabezpieczeń (fragmentacja, decoy, timing)

Znajomość popularnych narzędzi (jak nmap) pomaga w lepszym zrozumieniu perspektywy atakującego, ale sama wykrywalność opiera się na analizie zachowań sieciowych, nie sygnatur konkretnych programów.