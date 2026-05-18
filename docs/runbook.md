# Runbook operativo

## Setup iniziale

### 1. Verifica porta P1 (Sibelga - gia' attiva di default)

**Niente da richiedere a Sibelga.** Dalla policy ufficiale Sibelga:
"In Brussels, the P1 port is activated by default" (<https://www.sibelga.be/en/connections-meters/smart-meters/the-p1-port-of-the-smart-meter>).

Verifica fisica al contatore (~30 secondi):

1. Vai in cantina davanti al Landis+Gyr E360-1P.
2. Premi il pulsantino verde a fianco del display per ciclare le schermate.
3. Cerca l'indicatore `P1` o `GP` nel display LCD.
4. Sopra l'indicatore deve esserci una piccola **freccia accesa**: significa
   che la porta sta emettendo telegrammi DSMR ogni secondo.
5. Se la freccia non c'e' nonostante il default: contatta Sibelga via app My Sibelga
   o helpdesk (+32 2 274 40 66) per attivazione manuale - caso raro.

Nota: questo vale **solo per Sibelga (Brussels)**. Fluvius (Fiandre) e ORES (Vallonia)
richiedono richiesta esplicita via portale.

### 2. Pre-flight check sul laptop HP Spectre x360

Da fare prima di committarti definitivamente:

1. **Health SSD**: dal sistema corrente, lancia `sudo smartctl -a /dev/nvme0n1`
   (su Linux) o CrystalDiskInfo (su Windows).
   - Target: "Available Spare" >50%, "Percentage Used" <50%.
   - Se SSD cotto -> sostituisci con NVMe 2TB nuovo (~80 EUR) prima di tutto il resto.
2. **WiFi signal in cantina**: porta laptop in cantina e misura
   `nmcli dev wifi` (Linux) o "Network details" (Win).
   - Target: signal >= -65 dBm.
   - Se peggio: ordina mesh extender (~60 EUR) o powerline adapter Devolo (~50 EUR).
3. **Umidita' cantina**: target < 65%. Se piu' alta, deumidificatore portatile (~40 EUR).
4. **Batteria laptop**: se gonfia o degradata <30%, rimuovila fisicamente
   (4 viti sul fondo + connettore). Eviti rischi a lungo termine.
5. **Decisione su CMOS**: la batteria CR2032 della motherboard probabilmente e'
   scarica (causa il reset CMOS quando si stacca l'alimentatore). Soluzione:
   non staccare mai l'alimentatore. La sostituzione (~30 min smontaggio,
   batteria 1 EUR) e' opzionale.

### 3. BIOS setup (critico)

1. Boot del laptop, premi F10 (HP) per entrare in BIOS.
2. **Advanced -> Power Management -> AC Recovery / After Power Loss**: `Power On`
   (non "Last State" -> potrebbe restare spento dopo blackout).
3. **Boot Order**: NVMe interno primo, USB stick di install secondo.
4. **Secure Boot**: lascia abilitato se possibile (Fedora lo supporta).
5. Salva ed esci.

### 4. Installazione Fedora Server 44

1. Scarica ISO Fedora Server 44 da <https://fedoraproject.org/server/download/>.
2. Flash su chiavetta USB (>= 4 GB) con Fedora Media Writer, Balena Etcher,
   o `dd if=fedora.iso of=/dev/sdX bs=4M status=progress` (Linux).
3. Boot del laptop dalla chiavetta (F9 per boot menu HP).
4. Installer Anaconda:
   - **Software selection**: "Fedora Server Edition" + check "Headless Management"
     (per Cockpit) + "Virtualization Host" (per libvirt).
   - **Installation destination**: NVMe interno, "Custom layout" se vuoi
     un partizionamento controllato, altrimenti automatic (btrfs).
   - **Network & host name**:
     - Hostname: `nas-ha.local` (o quello che preferisci).
     - WiFi: connetti alla rete di casa, salva la connessione.
   - **Root password**: imposta.
   - **User**: crea utente non-root (es. `gio`) con sudo privileges.
5. Reboot, rimuovi chiavetta USB.
6. Login via console o `ssh gio@nas-ha.local`.

### 5. Hardening base e Cockpit

```bash
# Aggiorna tutto
sudo dnf upgrade --refresh -y

# Abilita Cockpit (gia' installato in Server edition)
sudo systemctl enable --now cockpit.socket

# Apri Cockpit nel firewall (solo zona interna)
sudo firewall-cmd --add-service=cockpit --permanent
sudo firewall-cmd --reload

# Plugin per gestire VM da Cockpit
sudo dnf install -y cockpit-machines

# Imposta timezone
sudo timedatectl set-timezone Europe/Brussels

# Verifica NTP
sudo systemctl status chronyd
```

Cockpit ora raggiungibile su <https://nas-ha.local:9090>.

### 6. WiFi stabilita' per uso 24/7

```bash
# Trova il nome della connessione
nmcli connection show

# Disabilita power save (causa drop su carichi 24/7)
sudo nmcli connection modify "<NomeWiFi>" wifi.powersave 2

# Retry infinito su disconnessione
sudo nmcli connection modify "<NomeWiFi>" connection.autoconnect-retries 0

# Preferenza banda 5 GHz (meno interferenza con PdC compressore)
sudo nmcli connection modify "<NomeWiFi>" wifi.band a   # 'a' = 5 GHz, 'bg' = 2.4 GHz

# Riavvia la connessione
sudo nmcli connection up "<NomeWiFi>"
```

### 7. Installa libvirt e prepara la VM HA OS

Gia' installato se hai selezionato "Virtualization Host" in fase install. Altrimenti:

```bash
sudo dnf install -y @virtualization libvirt-daemon-config-network virt-install virt-manager cockpit-machines
sudo systemctl enable --now libvirtd
sudo systemctl enable --now virtnetworkd
sudo usermod -aG libvirt,kvm $USER
sudo virsh net-autostart default
sudo virsh net-start default
# Rilogga per applicare il gruppo
exit
```

Verifica che il bridge virbr0 esista:

```bash
ip link show virbr0
# State DOWN e' normale finche' nessuna VM lo usa
```

### 7.bis Migrazione VM da Framework Desktop (preferito per Phase 2)

Se hai gia' una VM HA OS funzionante sul Framework Desktop (Phase 1), copia il disco qcow2
qui sullo Spectre invece di creare da zero. Cosi' mantieni tutte le configurazioni,
integrazioni, automazioni, state senza dover rifare tutto.

**Sul Framework**:

```bash
# Spegni la VM (per coerenza del disk)
sudo virsh shutdown homeassistant

# Aspetta che termini (~30s)
sudo virsh list --all | grep homeassistant
# Deve mostrare "shut off"

# Comprimi il qcow2 per trasferirla via USB stick / SCP / NAS
sudo qemu-img convert -O qcow2 -c -p \
  /var/lib/libvirt/images/haos.qcow2 \
  ~/haos-migration.qcow2
# (-c = compress, -p = progress)

# Trasferiscila sullo Spectre (opzione A: scp se sshd attivo sullo Spectre)
scp ~/haos-migration.qcow2 gio@<spectre-ip>:/tmp/

# Opzione B: copia su NAS, poi mount dallo Spectre
cp ~/haos-migration.qcow2 /mnt/nas/migration/

# Opzione C: USB stick
cp ~/haos-migration.qcow2 /run/media/gio/MyUSB/
```

**Sullo Spectre**:

```bash
# Sposta il qcow2 nella directory libvirt
sudo mv /tmp/haos-migration.qcow2 /var/lib/libvirt/images/haos.qcow2
sudo chown qemu:qemu /var/lib/libvirt/images/haos.qcow2

# Verifica
sudo qemu-img info /var/lib/libvirt/images/haos.qcow2

# Crea la VM importando il disco esistente
# IMPORTANTE: --boot uefi su Fedora 44 usa OVMF Secure Boot di default,
# che RIGETTA HA OS (non e' firmata Microsoft). Specifichiamo esplicitamente
# il firmware non-secure.
sudo virt-install \
  --name homeassistant \
  --vcpus 2 \
  --memory 4096 \
  --osinfo linux2022 \
  --disk /var/lib/libvirt/images/haos.qcow2,bus=virtio,format=qcow2 \
  --network network=default,model=virtio \
  --import \
  --boot loader=/usr/share/edk2/ovmf/OVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/edk2/ovmf/OVMF_VARS.fd \
  --graphics vnc,listen=127.0.0.1 \
  --noautoconsole

# Aspetta ~60s che HA OS riprenda lo stato
sleep 60
sudo virsh list --all

# Trova IP via DHCP libvirt default network
sudo virsh net-dhcp-leases default

# Set autostart (critico!)
sudo virsh autostart homeassistant

# Verifica
sudo virsh dominfo homeassistant | grep -E "State|Autostart"
```

**Riconfigura SSH key per Optima** (la chiave precedente puntava al Framework, ora gira sullo
Spectre con stesso indirizzo NAT):

```bash
# In HA via Studio Code Server -> Terminal
mkdir -p /root/.ssh
echo "<chiave-pubblica-optima>" > /root/.ssh/authorized_keys
chmod 600 /root/.ssh/authorized_keys
```

### 7.ter Fresh install (alternativa se non hai una VM da migrare)

```bash
cd /var/lib/libvirt/images
# Sostituisci <VERSION> con l'ultima release su:
#   https://github.com/home-assistant/operating-system/releases
sudo curl -L -o haos.qcow2.xz \
  https://github.com/home-assistant/operating-system/releases/download/<VERSION>/haos_ova-<VERSION>.qcow2.xz
sudo unxz haos.qcow2.xz
sudo chown qemu:qemu haos.qcow2
sudo qemu-img resize haos.qcow2 64G  # opzionale, default 32 GB

# Stessa virt-install della migrazione (con OVMF non-secure)
sudo virt-install \
  --name homeassistant \
  --vcpus 2 \
  --memory 4096 \
  --osinfo linux2022 \
  --disk /var/lib/libvirt/images/haos.qcow2,bus=virtio,format=qcow2 \
  --network network=default,model=virtio \
  --import \
  --boot loader=/usr/share/edk2/ovmf/OVMF_CODE.fd,loader.readonly=yes,loader.type=pflash,nvram.template=/usr/share/edk2/ovmf/OVMF_VARS.fd \
  --graphics vnc,listen=127.0.0.1 \
  --noautoconsole

sleep 60
sudo virsh net-dhcp-leases default
# Aspetta onboarding HA (~5 min al primo boot)
sudo virsh autostart homeassistant
```

### 8. Mount NAS Netgear come storage cold

```bash
sudo dnf install -y cifs-utils

# Credenziali in file protetto
sudo tee /root/.smbcred > /dev/null <<EOF
username=<utente nas>
password=<password nas>
EOF
sudo chmod 600 /root/.smbcred

# Mount point
sudo mkdir -p /mnt/nas

# Entry permanente in /etc/fstab
echo '//nas-34-18-FA.local/share /mnt/nas cifs credentials=/root/.smbcred,vers=3.0,uid=1000,gid=1000,iocharset=utf8,nofail,_netdev 0 0' \
  | sudo tee -a /etc/fstab

# Mount immediato
sudo mount /mnt/nas
ls /mnt/nas
```

`nofail` + `_netdev` significa: se il NAS non risponde all'avvio, il boot continua
comunque (importante per la resilienza). Il mount viene ritentato in background.

### 9. Reservation IP/DNS sul router

Sul router (Bbox / B-box / FRITZ!Box) riservare via MAC:

| Device | MAC | IP proposto |
|---|---|---|
| HP Spectre (WiFi) | (da `ip link show wlan0` su Fedora) | 192.168.129.10 |
| VM HA (bridge MAC) | (da `virsh domiflist homeassistant`) | 192.168.129.11 |
| HomeWizard P1 | (sticker o admin HomeWizard) | 192.168.129.12 |
| SMA Sunny Boy | (admin SMA) | 192.168.129.13 |
| Buderus MX300 | 00:60:34:3a:d3:82 | 192.168.129.15 (gia' impostato) |
| NAS Netgear | (admin nas) | 192.168.129.20 |

### 10. Integrazioni HA

#### HomeWizard

- Collegare il P1 alla porta del contatore via cavo RJ12 incluso.
- App HomeWizard Energy -> aggiungi dispositivo -> WiFi 2.4 GHz.
- HA: Settings -> Devices -> Add Integration -> "HomeWizard".
- Autodiscovery via mDNS dovrebbe trovarlo. Se no, immettere IP manualmente.
- Verifica entita': `sensor.p1_meter_active_power` deve esistere e variare.

#### SMA Solar

- HA: Settings -> Devices -> Add Integration -> "SMA Solar".
- Host: IP riservato. SSL: si (HTTPS). Group: "user". Password: quella SMA dell'app.
- Per dati installer (registri completi), chiedere a Brusol la password installer.

#### Buderus MX300 (HomeCom Easy)

- Installare HACS: <https://hacs.xyz/docs/use/download/download/>
- HACS -> Integrations -> Custom Repositories -> Aggiungi repo HomeCom Easy (es. <https://github.com/bujpe/bosch-homecom-easy> o equivalente attuale - cercare "HomeCom Easy" su HACS).
- Restart HA.
- Settings -> Devices -> Add Integration -> "Bosch HomeCom Easy".
- Credenziali: email/password dell'app My Home.
- Entita' attese:
  - `climate.<dispositivo>_central_heating`
  - `water_heater.<dispositivo>_hot_water`
  - sensori energy monitoring

### 11. Deploy delle configurazioni di questo repo

Opzione A (Studio Code Server addon):

1. Settings -> Add-ons -> Add-on Store -> "Studio Code Server" -> Install + Start.
2. Apri Studio Code Server (icona barra laterale).
3. Terminale integrato: `cd /config && git clone <repo>` (o aggiungere remote sul `/config` esistente).
4. Copia il contenuto di `configuration/` in `/config/configuration/`.
5. In `/config/configuration.yaml` aggiungi gli include:

   ```yaml
   homeassistant:
     packages: !include_dir_named configuration
   ```

6. Settings -> Developer Tools -> "Check Configuration" -> deve dare verde.
7. Restart HA.

Opzione B (Samba addon):

1. Installa Samba share addon.
2. Mount `/config` su workstation.
3. Copia file da repo.
4. Stesso step 5-7 di sopra.

### 12. Sostituzione entity_id placeholder

Tutti i file YAML hanno marker `# REPLACE: ...`. Cercarli ed adattarli:

```bash
grep -rn "REPLACE" /config/configuration/
```

Tipici sostituzioni:

| Placeholder | Da trovare in HA (Developer Tools -> States) |
|---|---|
| `sensor.sma_pv_power` | Cerca entita' che inizia per `sensor.sma_` con unita' W |
| `sensor.p1_meter_active_power` | Cerca `sensor.p1_` |
| `climate.heat_pump_central_heating` | Cerca `climate.` creato da HomeCom |
| `water_heater.heat_pump_hot_water` | Cerca `water_heater.` |
| `sensor.heat_pump_dhw_temperature` | Sensore temperatura serbatoio ACS della PdC |
| `sensor.outside_temperature` | Da PdC se esposto, altrimenti integrazione Meteo |
| `notify.mobile_app` | `notify.mobile_app_<device-name>` dopo install Companion App |

## Operazioni quotidiane

### Disattivare temporaneamente l'ottimizzatore

Toggle off `input_boolean.optimizer_enabled` dalla dashboard. Le automazioni di boost diventano no-op.

### Cambiare le soglie

Tutto via dashboard, sezione "Controlli ottimizzatore". Cambia in tempo reale, niente restart.

### Forzare boost ACS manuale

Developer Tools -> Services -> `water_heater.set_operation_mode` su `water_heater.heat_pump_hot_water` con mode "Boost".

## Troubleshooting

### Il sensore P1 e' "unavailable"

1. Verifica LED HomeWizard: deve essere verde fisso. Rosso = no connettivita' WiFi.
2. Ping IP HomeWizard dal Pi: `ping 192.168.129.11`.
3. Verifica che la porta P1 sia effettivamente attiva (freccia sopra `P1`/`GP` sul display del contatore). In Brussels e' attiva di default, ma una rara fault hardware puo' disattivarla.
4. App HomeWizard Energy: vedi dati live? Se no, problema con il P1, non con HA.
5. HA -> Settings -> Integrations -> HomeWizard -> Reload.

### `sensor.pv_surplus` resta a 0 anche con sole forte

1. Verifica che `sensor.p1_meter_active_power` vada negativo durante esportazione.
2. Se va positivo (= consumo) anche con FV alto = il P1 e' montato sul circuito sbagliato? Controllare cablaggio.
3. Developer Tools -> Template -> incolla template del template sensor per debug.

### La PdC non risponde ai comandi da HA

1. App My Home funziona ancora? Se no, problema lato cloud Bosch.
2. HA log: Settings -> System -> Logs -> cerca "homecom" o "bosch".
3. Reload integrazione HomeCom.
4. Verifica che la PdC non sia in modo "Manual" che blocca scritture remote (test pratico: prova a cambiare da app My Home prima).

### Cycling della PdC

Sintomo: la PdC parte e si ferma in continuazione perche' soglie troppo strette.
Fix:
- Aumenta `input_number.min_surplus_duration_minutes` (es. da 5 a 10).
- Aumenta `input_number.cooldown_minutes_between_boosts` (es. da 30 a 60).
- Aumenta isteresi: trigger di stop con soglia piu' bassa (modifica YAML).

## Acceptance test (post-installazione)

1. **Smart meter responsivo**: accendi forno (2 kW) -> `sensor.p1_meter_active_power` sale di ~2 kW entro 2 s.
2. **SMA accurato**: confronta `sensor.sma_pv_power` con app Sunny Portal -> delta < 5% in giornata di sole.
3. **PdC scrivibile**: cambia setpoint ACS da HA -> riflesso in app My Home entro 60 s.
4. **Surplus calc corretto**: in giornata di sole con solo frigo+standby, `sensor.pv_surplus` ~= produzione - 200 W.
5. **Automazione boost**: simula surplus (helper di test) per 5 min -> boost ACS parte -> setpoint cambia in app -> dopo stop, restore.
6. **Energy Dashboard nativa**: dopo 24h, kWh prod / cons / import / export chiudono entro 1-2%.

## Manutenzione

- Snapshot HA settimanali: Settings -> System -> Backups -> Auto-backup ON.
- Update HA: ogni 2 settimane controlla update, applica appena uscito dalle beta.
- Update integrazione HomeCom: HACS controlla update settimanali; leggere changelog prima di applicare (rischio breaking change su API Bosch).
- Pulizia log: HA gira un retention default di 10 giorni; per analisi piu' lunghe abilita InfluxDB addon.

## Contatti

| Cosa | Dove |
|---|---|
| Problema P1 | Sibelga: <https://www.sibelga.be/> |
| Problema FV | Brusol: <info@brusol.be> / +32 2 411 90 47 |
| Problema PdC | Buderus support BE / installatore originale |
| Bug integrazione HomeCom | Issue tracker GitHub del repo HACS usato |
