# Incident response cheatsheet 2026 by Matt "difr000" Sabaj

Niniejszy dokument stanowi przykład dobrych praktyk w dziedzinie Incident Response. 
Należy jednak pamiętać, że są to jedynie wytyczne lub zetstaw dobrych praktyk wpływających na profesjonalizację procesu DFIR.

## Wprowadzenie

W 2026 roku incident response nie może już ograniczać się do jednego hosta i jednego logu. Z materiałów Palo Alto Unit 42 wynika, że:

- ponad 750 dużych incydentów posłużyło do budowy trendów raportu
- 87% incydentów obejmowało więcej niż jedną powierzchnię ataku
- prawie 48% spraw zawierało komponent przeglądarkowy
- słabości tożsamości i uwierzytelniania odgrywały istotną rolę w niemal 90% analiz
- w ponad 90% naruszeń atakujący wykorzystali luki możliwe do ograniczenia: brak widoczności, nadmierne uprawnienia, niespójne logowanie, słabą segmentację i zbyt duże zaufanie wewnętrzne

Praktyczny wniosek jest prosty: pierwszy etap IR musi jednocześnie obejmować host, tożsamość, sieć, logi, infrastrukturę usługową, dostawców, wirtualizację, backupy i ślady w przeglądarkach. Ten cheatsheet ma prowadzić od pierwszego telefonu i zebrania wywiadu, przez zabezpieczenie materiału, po szybką analizę Windows i Linux/Unix.

## Zasady wspólne

```text
1. Evidence first, recovery second, chyba że życie/bezpieczeństwo lub ciągłość krytyczna wymaga inaczej.
2. Nie wyłączaj hosta odruchowo, jeżeli możesz jeszcze zebrać volatile evidence.
3. Izoluj logicznie, nie niszcząc RAM: VLAN, ACL, EDR isolation, odpięcie od segmentu.
4. Pracuj z jednego katalogu sprawy i zapisuj każdą czynność.
5. Zawsze haszuj artefakty i prowadź chain of custody.
6. Zabezpieczaj to, co rotuje się najszybciej: RAM, procesy, połączenia, logi lokalne, logi brzegowe.
7. Szukaj centralnych kolektorów logów i systemów kopii zanim zaczniesz działać lokalnie.
8. Zakładaj, że incydent może obejmować tożsamość, edge, RDP/VPN, SaaS, AD i backupy.
9. Traktuj backupy, logi uszkodzone i dane zaszyfrowane jako nadal wartościowe dowodowo.
10. Dziel działania na: wywiad, zabezpieczenie, analiza, containment, eradication, recovery.
```

## Szybki schemat decyzyjny

```text
START
1. Potwierdź zakres: host / user / serwer / edge / AD / backup / hypervisor / kilka segmentów
2. Oceń priorytet: evidence first vs recovery first
3. Ustal, które maszyny nadal pracują i czy warto chronić RAM
4. Zidentyfikuj logi lokalne, centralne i sieciowe
5. Ustal, czy działają backupy i czy nie nadpisują danych
6. Zabezpiecz dane ulotne i logi na systemach krytycznych
7. Zabezpiecz nośniki, snapshoty, obrazy i klucze odszyfrowujące
8. Zbuduj hipotezy: user / edge / VPN / web / RDP / usługa / konto uprzywilejowane
9. Zbuduj timeline i scope
10. Wdróż containment na hostach, kontach, segmentach i usługach
11. Przeprowadź eradication
12. Recovery + hardening + rotacja haseł/kluczy + poprawa telemetryki
END
```

## Artefakty obowiązkowe

```text
- data/czas z timezone
- hostname, FQDN, IP, MAC, OS/build/kernel, machine-id
- zalogowani użytkownicy i sesje
- procesy, parent process, cmdline, ścieżki, hashe, uchwyty, połączenia
- usługi, taski, startup, cron, timers, autoruns, GPO, Run/RunOnce
- konta lokalne i domenowe, grupy uprzywilejowane, sudo/admini
- firewall, ARP, routing, DNS, shares, RDP, SMB, WinRM, SSH, VPN
- logi: Security, System, Application, IIS, auth, syslog, journal, auditd, bramy, VPN, WAF, proxy, backup
- artefakty użytkownika: browser, downloads, jumplists, prefetch, shell history, RDP cache
- artefakty systemowe: MFT, UsnJrnl, SRUM, Amcache, NTUSER, UsrClass, SAM, SYSTEM, SOFTWARE, NTDS, PageFile
- RAM, obraz dysku lub snapshot, hashe, metadane obrazowania
```

## Powiadomienie o incydencie i pierwszy kontakt

### Co ustalić podczas pierwszej rozmowy

- Kiedy incydent został zauważony i przez kogo.
- Co przestało działać: usługi publiczne, poczta, pliki, domena, backup, hypervisor.
- Czy jakiekolwiek maszyny nadal pracują i nie były restartowane.
- Jakie działania własne już wykonano: restart, odłączenie sieci, skan AV, kasowanie plików, zmiana haseł.
- Czy istnieją logi centralne: SIEM, syslog, collector Windows Event Forwarding, logi firewall/VPN.
- Czy istnieje ryzyko rotacji logów lub nadpisania kopii.
- Czy środowisko obejmuje AD, VPN, usługi publiczne, wirtualizację, backup appliance, segmenty OT/IoT.
- Jakie dane mogły zostać dotknięte: osobowe, finansowe, medyczne, dane klientów, dane krytyczne państwowo.

### Macierz powiadomień i koordynacji

- `CERT Polska`: gdy skala, wektor lub wpływ uzasadniają wsparcie krajowe.
- `Policja / organy ścigania`: gdy doszło do włamania, szantażu, sabotażu lub wycieku.
- `UODO`: gdy istnieje ryzyko naruszenia danych osobowych.
- `CEZ / regulator branżowy / operator usług kluczowych`: gdy naruszenie dotyczy środowisk regulowanych.
- `Dostawcy i producenci`: gdy potrzebne są logi, snapshoty, recovery key, wsparcie do sprzętu lub chmury.
- `ISP / hosting / cloud`: gdy trzeba ustalić źródła ruchu, zablokować publikację danych lub zabezpieczyć telemetrię.
- `Zespół prawny i zarząd`: gdy potrzebne są decyzje o komunikacji zewnętrznej i tolerowanym ryzyku operacyjnym.

### Co zrobić w pierwszych 60 minutach

- Zachować spójny kanał koordynacji i właściciela incydentu.
- Zablokować samodzielne restarty i "sprzątanie" bez zgody zespołu IR.
- Wskazać systemy krytyczne do evidence-first.
- Wskazać systemy krytyczne do recovery-first.
- Zabezpieczyć centralne logi, backupy i edge.
- Ustalić kolejność zabezpieczania materiału.

## Zebranie wywiadu i ustalenie zakresu

### Informacje wstępne

Na podstawie lokalnej bazy wiedzy pierwsza faza wywiadu powinna objąć trzy obszary:

1. Aktualna sytuacja.
2. Infrastrukturę.
3. Rodzaj przetwarzanych danych i konsekwencje regulacyjne.

### Pytania o aktualną sytuację

- Co dokładnie widać: szyfrowanie, brak dostępu, alert EDR, nietypowe logowania, awaria usług.
- Co wykonali administratorzy i użytkownicy przed kontaktem z zespołem IR.
- Czy poszkodowany potrafi wskazać przybliżony wektor wejścia.
- Czy są maszyny nadal uruchomione.
- Czy pojawiły się noty okupu, nietypowe konta, nowe usługi, zadania, ruch wychodzący.

### Pytania o infrastrukturę instytucji

- Jakie są główne segmenty sieci i gdzie stoją systemy krytyczne.
- Jakie bramy wystawiają usługi do Internetu.
- Jak działa zdalny dostęp: VPN, RDP Gateway, publikacja usług, port forwarding.
- Czy środowisko ma AD, wiele domen, zaufania, oddziały i dostawców zdalnych.
- Jakie hypervisory działają: Hyper-V, VMware, KVM, Proxmox.
- Jak zorganizowano backupy, gdzie są repozytoria i kto ma do nich dostęp.
- Czy istnieją centralne kolektory logów systemowych i sieciowych.

### Pytania o dane i wpływ

- Jakiego typu dane mogły zostać naruszone.
- Czy środowisko należy do infrastruktury krytycznej lub współpracuje z instytucjami państwowymi.
- Czy istnieje obowiązek formalnego zgłoszenia i w jakim terminie.

## Infrastruktura instytucji: elementy obowiązkowe do sprawdzenia

### Bramy i urządzenia brzegowe

- Firewall, WAF, reverse proxy, router, brama VPN i tunele site-to-site to najczęstsze miejsca utraty widoczności.
- Brama jest jednocześnie punktem ochrony i bardzo atrakcyjnym celem ataku.
- Sprawdź konfiguracje, reguły, NAT, przekierowania portów, logi sesji i aktualność oprogramowania.

### Dostęp zdalny

- Usługi publiczne i port forwarding nie są zabezpieczeniem samym w sobie.
- Dostęp administracyjny z Internetu wymaga szczególnej ostrożności, bo zwykle łączy szerokie uprawnienia z dużym zasięgiem.
- Zbierz listę publikowanych usług, adresów, portów, kont administracyjnych i zasad MFA.

### Środowisko AD

- Przejęcie kontroli nad kontrolerem domeny oznacza praktycznie pełny wpływ na całą domenę.
- Zabezpiecz `NTDS`, `SYSTEM`, logi uwierzytelnienia, zmiany grup uprzywilejowanych, logowania RDP i ślady usług.
- Oceń relacje zaufania, konta serwisowe, delegacje, nadużycia uprawnień i ścieżki lateral movement.

### Wirtualizacja

- Hyper-V, VMware, KVM i Proxmox należy traktować jako warstwę krytyczną.
- Zbierz listę VM, snapshotów, datastore, hostów i kont administracyjnych.
- Atak na hypervisor lub system zarządzania może objąć wiele usług jednocześnie.

### Backupy

- Backupy są częścią incydentu, nie tylko mechanizmem odtworzeniowym.
- Nawet usunięte, uszkodzone lub częściowo zaszyfrowane backupy mogą nadal zawierać odzyskiwalne dane.
- Wstrzymaj procesy, które mogą nadpisywać niezaalokowaną przestrzeń lub dalsze wersje backupu.

### Dostawcy

- Zewnętrzni dostawcy mogą mieć logi, snapshoty, historię sesji, dane o ruchu i mechanizmy blokady udostępnionych danych.
- Już na starcie przygotuj listę wniosków do ISP, hostingu, producenta sprzętu, chmury i operatora backupu.

### Segmentacja

- Segmentacja fizyczna lub logiczna ogranicza blast radius i ułatwia containment.
- W IR trzeba ustalić nie tylko planowaną, ale faktyczną segmentację i rzeczywiste ścieżki komunikacji.

## Typowe hipotezy wektora wejścia

- użytkownik jako wektor wejścia: phishing, uruchomienie pliku, przejęcie przeglądarki, kradzież tokenu
- podatność w oprogramowaniu na urządzeniu brzegowym lub usłudze publicznej
- jednoskładnikowe uwierzytelnianie do VPN lub panelu administracyjnego
- przekierowanie portów do słabo chronionej usługi
- nadużycie legalnych poświadczeń i sesji

## Nośniki i obrazowanie

### Różnice dowodowe między nośnikami

#### HDD

- duża pojemność i niski koszt za GB
- wysoka wartość przy odzyskiwaniu usuniętych danych, jeśli nie zostały nadpisane
- wrażliwość mechaniczna i wolniejsze obrazowanie

#### SSD

- szybkie obrazowanie i dobra odporność w transporcie
- kluczowe ryzyko: `TRIM` może bezpowrotnie usunąć możliwość odzyskania skasowanych danych
- lepszy nośnik do szybkiego zabezpieczenia, gorszy do liczenia na odzysk kasowanych plików

#### M.2 / NVMe / SATA / mSATA / PCIe / U.2 / SAS / IDE

- przed podłączeniem ustal typ interfejsu, kluczowanie, zasilanie i zgodność adaptera
- `M.2` może oznaczać zarówno `SATA`, jak i `NVMe`; samo gniazdo nie daje pewności kompatybilności
- `SAS` i `SATA` są łatwe do pomylenia operacyjnie, ale nie są w pełni zamienne
- starsze `IDE/PATA` wciąż mogą występować w starszych systemach i sprzęcie przemysłowym

#### Karty pamięci

- `SD`, `CFast`, `CFexpress` często występują w urządzeniach mobilnych, kamerach, dronach, urządzeniach przemysłowych
- wymagają własnych czytników i szczególnej ostrożności przy zabezpieczaniu adapterów

### Zasady obrazowania

```text
1. Zawsze używaj blokera zapisu lub sprzętu gwarantującego read-only.
2. Najpierw zidentyfikuj nośnik źródłowy i docelowy.
3. Sprawdź typ interfejsu i potrzebne adaptery.
4. Zweryfikuj pojemność i wolne miejsce na nośniku docelowym.
5. Dokumentuj model, numer seryjny, interfejs i fizyczny kontekst nośnika.
6. Preferuj format E01/EWF z metadanymi i weryfikacją.
7. Licz hash obrazu i pakietu wynikowego.
8. Jeśli nośnik jest szyfrowany, zabezpiecz klucze odzyskiwania zanim odłączysz system.
```

### Narzędzia do obrazowania i live response

- `Tableau TX1`: sprzętowy write blocker i imager do pracy terenowej
- `ewf-tools`: linuksowe obrazowanie i weryfikacja `E01`
- `Guymager`: wygodne obrazowanie z dystrybucji live
- `CAINE`, `Tsurugi`, `Paladin`: dystrybucje live przydatne do akwizycji i triage
- `Ventoy`: przygotowanie wielosystemowego nośnika USB z wieloma obrazami ISO

### Minimalna procedura Linux live / ewf-tools

```bash
lsblk
fdisk -l
mkdir -p /mnt/obrazowanie/src /mnt/obrazowanie/dst
mount -o ro,show_sys_files,streams_interface=windows /dev/sdXN /mnt/obrazowanie/src
df -h
sudo ewfacquire -t /mnt/obrazowanie/dst/host_disk /dev/sdX
ewfinfo /mnt/obrazowanie/dst/host_disk.E01
ewfverify /mnt/obrazowanie/dst/host_disk.E01
sha256sum /mnt/obrazowanie/dst/host_disk.E01
```

## Analiza: najważniejsze narzędzia i zastosowania

### Narzędzia do inspekcji i konwersji artefaktów Windows

- `EvtxECmd`: konwersja `EVTX` do `CSV/JSON/XML`; kluczowe do szybkiego triage logów i map eventów
- `Timeline Explorer`: filtrowanie i analiza wyników `CSV` z narzędzi Zimmermana i innych źródeł
- `Registry Explorer`: analiza `SYSTEM`, `SOFTWARE`, `SAM`, `NTUSER.dat`, `UsrClass.dat`
- `AmcacheParser`: konwersja `Amcache.hve`
- `PECmd`: analiza `Prefetch`
- `JLECmd`: analiza `Jump Lists`
- `MFTECmd`: analiza `$MFT`, `$UsnJrnl`, `$LogFile`, `$Boot`, `$I30`
- `SrumECmd`: analiza `SRUDB.dat`
- `SumECmd`: analiza `User Access Logging`
- `ntdsextract2`: ekstrakcja użytkowników, grup, komputerów i timeline z `ntds.dit`
- `DB Browser for SQLite`: przegląd baz SQLite, np. historii przeglądarek
- `EseDatabaseView`: szybki podgląd baz `ESE`
- `BrowsingHistoryView` i `BrowserDownloadsView`: historia przeglądania i pobrań
- `bmc-tools` + `RdpCacheStitcher`: ekstrakcja i składanie pamięci podręcznej klienta RDP
- `bulk_extractor`: odzysk i ekstrakcja danych z obrazów, w tym śladów z niezaalokowanej przestrzeni
- `Mimikatz`: analiza zrzutów `LSASS` i poświadczeń, jeżeli taki dump został odnaleziony

### Narzędzia i techniki przydatne w Linux/Unix

- `grep` i `ripgrep`: szybkie przeszukiwanie logów, konfiguracji i dumpów tekstowych
- `journalctl`: analiza `systemd-journal`
- `ausearch` i `aureport`: analiza `auditd`
- `ps`, `lsof`, `ss`, `netstat`, `ip`, `find`, `sha1sum`, `systemctl`, `crontab`
- `logrotate`, `rsyslog`, `journald.conf`: weryfikacja retencji, rotacji i kolektorów zdalnych

## Analiza: artefakty szczególnie cenne

### Windows

- `C:\Windows\System32\winevt\Logs`
- `C:\inetpub\logs\LogFiles`
- `C:\Windows\AppCompat\Programs\Amcache.hve`
- `C:\Windows\Prefetch`
- `C:\Windows\System32\sru\SRUDB.dat`
- `C:\Windows\System32\LogFiles\sum`
- `C:\Windows\NTDS\ntds.dit`
- `C:\PageFile.sys`
- `C:\Users\<user>\NTUSER.dat`
- `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\AutomaticDestinations`
- `C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\CustomDestinations`
- `C:\Users\<user>\AppData\Local\Microsoft\Terminal Server Client\Cache`

### Linux/Unix

- `/var/log/`
- `/var/log/journal/`
- `/var/log/audit/audit.log`
- `/etc/rsyslog.conf`, `/etc/rsyslog.d/*`, `/etc/syslog.conf`
- `/etc/systemd/journald.conf`, `/etc/systemd/journal-upload.conf`
- `/etc/crontab`, `/etc/cron*`, `/var/spool/cron`
- `/etc/systemd/system`, `/usr/lib/systemd/system`
- `/home/*/.bash_history`, `/root/.bash_history`
- `/home/*/.ssh/authorized_keys`, `/root/.ssh/authorized_keys`
- `/home/*/.local/share/recently-used.xbel`

## Linux / Unix: algorytm od początku do końca

### 1. Przygotuj katalog sprawy

```bash
sudo -i
export CASE=IR_$(hostname)_$(date +%F_%H%M%S)
export IR_DIR=/root/$CASE
mkdir -p "$IR_DIR"/{live,logs,network,processes,persistence,fs,users,web,hashes,system,users_export}
exec > >(tee -a "$IR_DIR/live/console.log") 2>&1
date -Ins
hostnamectl
uname -a
```

### 2. Zabezpiecz podstawowe informacje o systemie

```bash
hostname | tee "$IR_DIR/system/hostname.txt"
uname -a | tee "$IR_DIR/system/uname.txt"
cat /etc/*release* 2>/dev/null | tee "$IR_DIR/system/os-release.txt"
cat /etc/hostname 2>/dev/null | tee "$IR_DIR/system/etc-hostname.txt"
cat /etc/passwd | tee "$IR_DIR/system/passwd.txt"
cat /etc/group | tee "$IR_DIR/system/group.txt"
ip a | tee "$IR_DIR/network/ip_a.txt"
ip route | tee "$IR_DIR/network/ip_route.txt"
ip neigh | tee "$IR_DIR/network/ip_neigh.txt"
cat /etc/resolv.conf | tee "$IR_DIR/network/resolv.conf.txt"
cat /etc/hosts | tee "$IR_DIR/network/hosts.txt"
```

### 3. Dane ulotne i procesy

```bash
date -Ins | tee "$IR_DIR/live/date.txt"
uptime | tee "$IR_DIR/live/uptime.txt"
who -a | tee "$IR_DIR/users/who-a.txt"
w | tee "$IR_DIR/users/w.txt"
last -a | head -200 | tee "$IR_DIR/users/last.txt"
lastlog | tee "$IR_DIR/users/lastlog.txt"
ps auxwwf | tee "$IR_DIR/processes/ps_auxwwf.txt"
pstree -ap | tee "$IR_DIR/processes/pstree_ap.txt"
find -L /proc/[0-9]*/exe -print0 2>/dev/null | xargs -0 sha1sum 2>/dev/null | tee "$IR_DIR/processes/process_sha1.txt"
find /proc/[0-9]*/cmdline | xargs head 2>/dev/null | tee "$IR_DIR/processes/process_cmdline.txt"
lsof +c0 -M -R -V -w -n -P | tee "$IR_DIR/processes/process_opened_files.txt"
```

### 4. Sieć, sesje i aktywne połączenia

```bash
ss -pantul | tee "$IR_DIR/network/ss_pantul.txt"
ss -panto state established | tee "$IR_DIR/network/ss_established.txt"
lsof -nP -i | tee "$IR_DIR/network/lsof_i.txt"
netstat -nap 2>/dev/null | tee "$IR_DIR/network/netstat_nap.txt"
arp -a | tee "$IR_DIR/network/arp.txt"
iptables -L -n -v | tee "$IR_DIR/network/iptables_Lnv.txt"
iptables -S | tee "$IR_DIR/network/iptables_S.txt"
nft list ruleset | tee "$IR_DIR/network/nft_ruleset.txt" 2>/dev/null
```

Szukaj od razu:

```bash
grep -E '(:4444|:5555|:8080|:8443|:1337|:9001|:53 |:443 )' "$IR_DIR/network/ss_pantul.txt"
grep -E 'nc |ncat |socat |bash -i|sh -i|/dev/tcp/|curl |wget |python -c|perl -e|php -r|ruby -e' /proc/*/cmdline 2>/dev/null
```

### 5. Logi i systemy logowania

#### Jeśli system używa `systemd`

```bash
journalctl --no-pager -b | tee "$IR_DIR/logs/journal_boot.txt"
journalctl --no-pager --since '7 days ago' | tee "$IR_DIR/logs/journal_7d.txt"
cat /etc/systemd/journald.conf 2>/dev/null | tee "$IR_DIR/logs/journald.conf.txt"
cat /etc/systemd/journal-upload.conf 2>/dev/null | tee "$IR_DIR/logs/journal-upload.conf.txt"
cat /etc/machine-id 2>/dev/null | tee "$IR_DIR/system/machine-id.txt"
```

#### Jeśli system używa `syslog/rsyslog`

```bash
ls -la /var/log | tee "$IR_DIR/logs/varlog_listing.txt"
cat /etc/rsyslog.conf 2>/dev/null | tee "$IR_DIR/logs/rsyslog.conf.txt"
cat /etc/syslog.conf 2>/dev/null | tee "$IR_DIR/logs/syslog.conf.txt"
find /etc/rsyslog.d -type f -maxdepth 1 -exec sed -n '1,200p' {} \; 2>/dev/null | tee "$IR_DIR/logs/rsyslog_d.txt"
cat /etc/logrotate.conf 2>/dev/null | tee "$IR_DIR/logs/logrotate.conf.txt"
find /etc/logrotate.d -type f -maxdepth 1 -exec sed -n '1,200p' {} \; 2>/dev/null | tee "$IR_DIR/logs/logrotate_d.txt"
```

#### Jeśli aktywny jest `auditd`

```bash
cat /etc/audit/audit.rules 2>/dev/null | tee "$IR_DIR/logs/audit.rules.txt"
ausearch -ts recent -m USER_LOGIN,USER_AUTH,ADD_USER,DEL_USER,EXECVE,SERVICE_START,SERVICE_STOP 2>/dev/null | tee "$IR_DIR/logs/audit_recent.txt"
aureport -au 2>/dev/null | tee "$IR_DIR/logs/aureport_auth.txt"
```

### 6. Persistence, startup i użytkownicy

```bash
systemctl list-units --type=service --all | tee "$IR_DIR/persistence/systemctl_units.txt"
systemctl list-unit-files | tee "$IR_DIR/persistence/systemctl_unit_files.txt"
systemctl list-timers --all | tee "$IR_DIR/persistence/systemd_timers.txt"
cat /etc/crontab | tee "$IR_DIR/persistence/crontab.txt"
for u in $(cut -d: -f1 /etc/passwd); do crontab -u "$u" -l 2>/dev/null | sed "s#^#[$u] #"; done | tee "$IR_DIR/persistence/user_crontabs.txt"
ls -la /etc/cron* /var/spool/cron /var/spool/cron/crontabs 2>/dev/null | tee "$IR_DIR/persistence/cron_dirs.txt"
cat /etc/sudoers | tee "$IR_DIR/users/sudoers.txt"
ls -la /etc/sudoers.d | tee "$IR_DIR/users/sudoers_d.txt"
find /root /home -maxdepth 3 -type f \( -name authorized_keys -o -name known_hosts -o -name .bash_history \) -exec ls -la {} \; -exec sed -n '1,200p' {} \; | tee "$IR_DIR/users/user_key_material.txt"
```

### 7. Pliki, ostatnie zmiany i webroot

```bash
find / -xdev -type f -mtime -7 -printf '%TY-%Tm-%Td %TT %u %g %m %s %p\n' 2>/dev/null | sort | tee "$IR_DIR/fs/recent_files_7d.txt"
find /tmp /var/tmp /dev/shm -maxdepth 3 -type f -printf '%TY-%Tm-%Td %TT %u %g %m %s %p\n' 2>/dev/null | sort | tee "$IR_DIR/fs/tmp_files.txt"
find / -xdev -perm -4000 -type f -print 2>/dev/null | tee "$IR_DIR/fs/suid.txt"
getcap -r / 2>/dev/null | tee "$IR_DIR/fs/file_capabilities.txt"
grep -Rni 'LD_PRELOAD\|LD_LIBRARY_PATH' /etc /root /home 2>/dev/null | tee "$IR_DIR/fs/ld_preload_hits.txt"
find /var/www /srv/www /usr/share/nginx/html /var/www/html /opt -type f -printf '%TY-%Tm-%Td %TT %s %p\n' 2>/dev/null | sort | tee "$IR_DIR/web/web_files.txt"
grep -RniE 'eval\\(|base64_decode\\(|assert\\(|system\\(|shell_exec\\(|passthru\\(|exec\\(|/bin/sh|/bin/bash|python -c|perl -e|gzinflate\\(' /var/www /srv/www /usr/share/nginx/html /var/www/html 2>/dev/null | tee "$IR_DIR/web/webshell_hits.txt"
```

### 8. Obrazowanie i pakowanie

```bash
fdisk -l | tee "$IR_DIR/fs/fdisk_l.txt"
lsblk -a | tee "$IR_DIR/fs/lsblk.txt"
blkid | tee "$IR_DIR/fs/blkid.txt"
find "$IR_DIR" -type f -exec sha256sum {} \; | tee "$IR_DIR/hashes/sha256_all.txt"
tar -czf "/root/${CASE}.tgz" "$IR_DIR"
sha256sum "/root/${CASE}.tgz" | tee "/root/${CASE}.tgz.sha256"
```

### Linux / Unix: czego szukać konkretnie

```text
- reverse shell: nc, ncat, socat, bash -i, python -c, perl -e
- nietypowy outbound do Internetu z serwera, który zwykle nie wychodzi
- nowe klucze SSH i nowe wpisy w authorized_keys
- nowa usługa systemd, timer, cron lub wpis w shell profile
- proces z usuniętym plikiem, memfd, nietypowymi capabilities lub LD_PRELOAD
- logi wysyłane do zdalnego kolektora, którego wcześniej nie znano
- ślady exfilu w /var/log, webserverach, tunelach, scp/rsync/curl/wget
```

## Windows / Windows Server: algorytm od początku do końca

### 1. Priorytety zabezpieczenia

Najważniejsze artefakty według lokalnej bazy wiedzy:

- dane ulotne z uruchomionej maszyny
- dzienniki `EVTX`
- logi `IIS`
- klucz odzyskiwania `BitLocker`
- `Amcache.hve`
- `SAM`, `SYSTEM`, `SOFTWARE`, `SECURITY`
- `NTUSER.dat` i `UsrClass.dat`
- `Jump Lists`
- `Prefetch`
- `SRUDB.dat`
- `$MFT` i `$UsnJrnl`
- `NTDS.dit` na kontrolerach domeny

### 2. Przygotuj katalog sprawy

```powershell
$Case="IR_$env:COMPUTERNAME"+"_"+(Get-Date -Format "yyyy-MM-dd_HHmmss")
$IR="C:\$Case"
New-Item -ItemType Directory -Force -Path $IR,$IR\live,$IR\users,$IR\proc,$IR\net,$IR\persistence,$IR\logs,$IR\fs,$IR\web | Out-Null
Start-Transcript -Path "$IR\live\transcript.txt" -Force
Get-Date -Format o | Tee-Object "$IR\live\date.txt"
hostname | Tee-Object "$IR\live\hostname.txt"
systeminfo | Out-File "$IR\live\systeminfo.txt"
```

### 3. Dane ulotne, użytkownicy, procesy

```cmd
whoami /all > "%IR%\users\whoami_all.txt"
query user > "%IR%\users\query_user.txt"
quser > "%IR%\users\quser.txt"
qwinsta > "%IR%\users\qwinsta.txt"
net user > "%IR%\users\net_user.txt"
net localgroup administrators > "%IR%\users\local_admins.txt"
tasklist > "%IR%\proc\tasklist.txt"
tasklist /svc > "%IR%\proc\tasklist_svc.txt"
wmic process get ProcessId,ParentProcessId,Name,ExecutablePath,CommandLine /format:list > "%IR%\proc\wmic_process.txt"
netstat -ano > "%IR%\net\netstat_ano.txt"
arp -a > "%IR%\net\arp.txt"
route print > "%IR%\net\route_print.txt"
ipconfig /all > "%IR%\net\ipconfig_all.txt"
```

```powershell
Get-ComputerInfo | Out-File "$IR\live\Get-ComputerInfo.txt"
Get-LocalUser | Format-List * | Out-File "$IR\users\Get-LocalUser.txt"
Get-CimInstance Win32_Process | Select-Object Name,ProcessId,ParentProcessId,ExecutablePath,CommandLine,CreationDate | Format-List | Out-File "$IR\proc\Win32_Process.txt"
Get-NetTCPConnection | Sort-Object LocalPort | Format-Table -Auto | Out-File "$IR\net\Get-NetTCPConnection.txt"
Get-WinEvent -LogName 'Microsoft-Windows-Windows Firewall With Advanced Security/Firewall' -MaxEvents 500 | Format-List | Out-File "$IR\logs\Firewall_500.txt"
```

### 4. Dzienniki zdarzeń i logi IIS

```cmd
wevtutil epl Security "%IR%\logs\Security.evtx"
wevtutil epl System "%IR%\logs\System.evtx"
wevtutil epl Application "%IR%\logs\Application.evtx"
```

```powershell
Get-WinEvent -LogName Security -MaxEvents 1000 | Format-List | Out-File "$IR\logs\Security_1000.txt"
Get-WinEvent -LogName System -MaxEvents 1000 | Format-List | Out-File "$IR\logs\System_1000.txt"
Get-WinEvent -LogName Application -MaxEvents 1000 | Format-List | Out-File "$IR\logs\Application_1000.txt"
Get-ChildItem 'C:\inetpub\logs\LogFiles' -Recurse -ErrorAction SilentlyContinue | Select-Object FullName,LastWriteTime,Length | Out-File "$IR\web\iis_log_inventory.txt"
```

### 5. Persistence i artefakty użytkownika

```cmd
sc query state= all > "%IR%\persistence\sc_query_all.txt"
schtasks /query /fo LIST /v > "%IR%\persistence\schtasks.txt"
reg query HKLM\SOFTWARE\Microsoft\Windows\CurrentVersion\Run > "%IR%\persistence\run_hklm.txt"
reg query HKCU\SOFTWARE\Microsoft\Windows\CurrentVersion\Run > "%IR%\persistence\run_hkcu.txt"
dir /s /b "C:\ProgramData\Microsoft\Windows\Start Menu\Programs\Startup" > "%IR%\persistence\startup_programdata.txt"
```

```powershell
Get-CimInstance Win32_Service | Select-Object Name,State,StartMode,ProcessId,PathName | Format-List | Out-File "$IR\proc\Win32_Service.txt"
Get-ScheduledTask | Select-Object TaskName,TaskPath,State,Author,Description | Format-List | Out-File "$IR\persistence\Get-ScheduledTask.txt"
Get-CimInstance Win32_StartupCommand | Select-Object Name,Command,Location,User | Format-List | Out-File "$IR\persistence\Win32_StartupCommand.txt"
```

### 6. Artefakty dyskowe Windows do natychmiastowego kopiowania

```text
EVTX:
C:\Windows\System32\winevt\Logs

IIS:
C:\inetpub\logs\LogFiles

Amcache:
C:\Windows\AppCompat\Programs\Amcache.hve

Prefetch:
C:\Windows\Prefetch

SRUM:
C:\Windows\System32\sru

UAL:
C:\Windows\System32\LogFiles\sum

Jump Lists:
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\AutomaticDestinations
C:\Users\<user>\AppData\Roaming\Microsoft\Windows\Recent\CustomDestinations

RDP cache:
C:\Users\<user>\AppData\Local\Microsoft\Terminal Server Client\Cache

NTUSER / UsrClass:
C:\Users\<user>\NTUSER.dat
C:\Users\<user>\AppData\Local\Microsoft\Windows\UsrClass.dat

Rejestry systemowe:
HKLM\SAM
HKLM\SYSTEM
HKLM\SOFTWARE
HKLM\SECURITY

AD:
C:\Windows\NTDS\ntds.dit
```

### 7. BitLocker i obrazowanie

```powershell
Get-BitLockerVolume | Format-List * | Out-File "$IR\live\Get-BitLockerVolume.txt"
manage-bde -status > "$IR\live\manage-bde-status.txt"
manage-bde -protectors -get C: > "$IR\live\manage-bde-protectors-C.txt"
```

Praktycznie:

- przed odłączeniem lub obrazowaniem nośnika zabezpiecz recovery key
- przy hostach szyfrowanych potraktuj recovery key jako artefakt krytyczny
- na kontrolerach domeny i serwerach wirtualizacyjnych sprawdź też klucze backupów i datastore

### 8. Sugerowana analiza artefaktów Windows

```text
EVTX -> EvtxECmd -> Timeline Explorer
Amcache.hve -> AmcacheParser -> Timeline Explorer
Prefetch -> PECmd -> Timeline Explorer
Jump Lists -> JLECmd -> Timeline Explorer
SRUM -> SrumECmd -> Timeline Explorer
UAL -> SumECmd -> Timeline Explorer
NTDS -> ntdsextract2 -> Timeline Explorer / analiza relacji
RDP cache -> bmc-tools -> RdpCacheStitcher
Browser / Downloads -> BrowsingHistoryView / BrowserDownloadsView / DB Browser for SQLite
NTUSER / UsrClass / SYSTEM / SOFTWARE / SAM -> Registry Explorer
$MFT / $UsnJrnl -> MFTECmd -> Timeline Explorer
```

### 9. Windows: czego szukać konkretnie

```text
- udane i nieudane logowania 4624, 4625, 4648, 4672
- instalacja usług 4697 i 7045
- zadania 4698 i 4702
- czyszczenie logów 1102
- uruchomienia procesów 4688 i Sysmon 1
- DNS, RDP, SMB i Defender z logów operacyjnych
- PowerShell, WMI, Task Scheduler, Sysmon
- nowe wpisy Run/RunOnce, scheduled tasks, usługi z Temp/AppData/ProgramData
- ślady uruchomień z Prefetch, Amcache, Jump Lists, UserAssist, RecentDocs, SRUM
- ślady RDP w logach i cache klienta
```

## Containment, eradication i recovery

### Containment po zebraniu dowodów

- blokada kont i tokenów
- wyłączenie lub izolacja hostów tylko po zabezpieczeniu najcenniejszych danych
- blokada IP, domen, portów i tuneli C2
- segmentacja awaryjna i odcięcie podatnych usług publicznych
- wycofanie dostępu dostawców i kont serwisowych bez ownera

### Eradication

- usuń webshell, usługę, task, wpis startup, klucz, konto, tunel, regułę, implant
- przywróć poprawną konfigurację edge, VPN, RDP, IIS i GPO
- usuń nadmierne uprawnienia i stare ścieżki zaufania
- napraw retencję i forwarding logów

### Recovery

- rotacja haseł, kluczy SSH, haseł lokalnych, poświadczeń serwisowych, recovery secrets
- przywrócenie MFA i kontroli dostępu uprzywilejowanego
- odtworzenie z backupów po ich walidacji i sprawdzeniu czystości
- wzbogacenie telemetryki host, edge, AD, proxy, SaaS, browser
- kontrola browser-based risk i tożsamości, zgodnie z trendami z raportu Palo Alto

## Lessons learned: co poprawić po incydencie

- centralizacja logów i spójna retencja
- lepsza segmentacja i widoczność ruchu east-west
- weryfikacja zdalnego dostępu i publikacji usług
- twarde zarządzanie tożsamością i least privilege
- procedury dla hypervisorów i backupów
- gotowe skrypty i nośniki live do akwizycji Windows/Linux
- katalog narzędzi i adapterów do nośników: SATA, SAS, M.2, NVMe, U.2, IDE, karty pamięci

## Krótka checklista końcowa

```text
[ ] ustalono wektor lub hipotezy wejścia
[ ] zabezpieczono logi lokalne, centralne i sieciowe
[ ] zabezpieczono volatile evidence na systemach pracujących
[ ] zabezpieczono artefakty użytkownika i systemowe
[ ] zabezpieczono backupy oraz warstwę wirtualizacji
[ ] policzono hashe i opisano chain of custody
[ ] zbudowano timeline i scope
[ ] wdrożono containment dla hostów, kont i segmentów
[ ] przygotowano plan eradication i recovery
[ ] zapisano lesson learned oraz działania naprawcze
```
