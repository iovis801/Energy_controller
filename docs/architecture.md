# Architettura

## Obiettivo

Ridurre la spesa energetica massimizzando l'auto-consumo del fotovoltaico in una casa belga senza batteria di accumulo. In Belgio l'iniezione di surplus in rete viene compensata a tariffe punitive: l'unica leva e' spostare i carichi nelle ore di sole.

## Hardware

| Componente | Modello | Connessione |
|---|---|---|
| Smart meter | Landis+Gyr E360-1P (Sibelga) | Porta P1-HAN RJ12 (DSMR-like) |
| Inverter FV | SMA Sunny Boy (Brusol) | WiFi (Speedwire/Modbus locale + cloud Sunny Portal) |
| Pompa di calore | Buderus MX300 | WiFi via gateway K30RF v1 (192.168.129.15), HomeCom Easy cloud |
| Lettore P1 | HomeWizard Wi-Fi P1 Meter | WiFi 2.4 GHz, alimentato da P1 |
| Host | HP Spectre x360 (i7-8705G, 20 GB DDR4, 512 GB NVMe) | WiFi, in cantina, sempre alimentato |
| Storage cold | NAS Netgear esistente | SMB su rete WiFi locale, montato su `/mnt/nas` dal host |

## Stack software

### Host (HP Spectre x360 in cantina)

- **Fedora Server 44**: kernel recente, Cockpit web UI nativa (porta 9090), btrfs di default.
- **libvirt + qemu/kvm**: hypervisor leggero per la VM Home Assistant OS.
- **cockpit-machines**: plugin Cockpit per gestire le VM via browser.
- **podman** (e/o Docker): runtime container per i futuri agent worker (repo separato `loki-workers/`).
- **systemd timers**: scheduler nativo per cron-like (preferito a cron classico per coerenza/logging).
- **Tailscale**: accesso remoto sicuro al host + a Cockpit + a HA UI senza aprire porte sul router.
- **NetworkManager**: profilo WiFi con `connection.autoconnect-retries=0` (riconnessione infinita)
  e `wifi.powersave=2` (no power save) per stabilita' del link 24/7.
- **SMB client**: monta condivisione del NAS Netgear su `/mnt/nas` via `cifs-utils` + entry in `/etc/fstab`.

### Guest VM Home Assistant

- **Home Assistant OS** x86-64 (immagine `.qcow2` da home-assistant.io).
- 2 vCPU, 4 GB RAM, 32 GB disk (su SSD locale via virtio).
- Network in modalita' bridge -> IP dedicato sulla LAN, raggiungibile direttamente.
- **HACS** per integrazioni community (HomeCom Easy).
- **InfluxDB + Grafana** addon (opzionale): solo se serve storico > 10 giorni.
- **Studio Code Server** addon: editor in-browser per sync con questo repo.

## Flusso dati

```
HomeWizard P1 ----(mDNS push)----> HA: sensor.p1_meter_active_power (W, segno: + import, - export)
SMA Sunny Boy ----(Modbus/Speedwire poll)----> HA: sensor.sma_pv_power (W)
Buderus MX300 ----(HomeCom cloud poll)----> HA: climate / water_heater / sensori
                                              <----(scrittura setpoint via cloud)

HA template sensors:
  home_consumption = sma_pv_power + p1_active_power     # (export e' negativo => sottrae)
  pv_surplus       = max(0, -p1_active_power)            # quanto sto iniettando
  pv_surplus_avg_5min  = media mobile 5 min di pv_surplus
  pv_surplus_avg_15min = media mobile 15 min di pv_surplus

HA automations:
  surplus_avg_5min > soglia_acs per 5 min => boost ACS a 60 C
  surplus_avg_15min > soglia_heat per 10 min => +1.5 C setpoint riscaldamento
  surplus_avg_15min > soglia_notify per 15 min => push "buon momento per lavatrice"
```

## Decisioni architetturali

### Perche' Home Assistant e non custom

- Energy Dashboard nativa gia' pronta.
- Integrazioni HomeWizard e SMA ufficiali, manutenute.
- Buderus MX300/K30RF e' cloud-only (HomeCom Easy): scrivere un client Python custom richiederebbe reverse-engineering dell'API Bosch. Integrazione community in HA fa gia' questo lavoro.
- Time-to-value: 1 weekend vs 2-4 settimane custom.

### Perche' HomeWizard P1 e non un DSMR USB

- Plug-and-play, certificato per il P1 belga/olandese.
- Alimentato dalla porta P1: zero cavi extra.
- Integrazione HA ufficiale (non community).
- Espone anche un endpoint HTTP locale (utile per debugging fuori HA).
- Costa ~120 EUR ma risparmia ore di setup.

### Perche' HP Spectre x360 riusato in cantina

- **Costo zero**: hardware gia' di proprieta', non in uso.
- **CPU sovradimensionata**: i7-8705G (4c/8t) gira HA + agent containers + qualche servizio
  ancillare senza pensieri. Piu' potente di qualsiasi NAS appliance sotto i 600 EUR.
- **Posizione termica ideale**: cantina fresca a 12-15 C tiene la CPU rilassata
  anche sotto carico (il chip Kaby Lake-G e' notoriamente caldo, in soggiorno andrebbe a 80 C+).
- **Vicinanza alla PdC**: corto cammino se in futuro si vorra' aprire l'opzione EMS-ESP locale.
- **Form factor laptop come "UPS gratis"**: la batteria interna (anche se degradata)
  fa da buffer per blackout brevi di rete elettrica. Combinato con BIOS "AC Recovery: Power On",
  il sistema si auto-riprende.
- **Rumore zero in casa**: ventola in cantina = inudibile.

### Perche' Fedora Server e non Proxmox

- Linux general-purpose vs hypervisor specializzato. Volevamo flessibilita' anche
  per ospitare agenti AI (Claude Code, Python services) accanto a HA, non solo VM.
- Cockpit copre il 90% dei casi d'uso di Proxmox web UI senza imporre paradigmi.
- Repository Fedora ricco: podman, libvirt, gh, claude, node, python, tailscale,
  tutto installabile con `dnf install`.
- Update annuale (Fedora) accettabile per uso casalingo.

### Perche' WiFi-only e non ethernet

- Tutto l'ecosistema downstream e' gia' WiFi (HomeWizard, SMA, Buderus).
- Aggiungere ethernet solo per il host non migliora la robustezza end-to-end.
- Cablare la cantina costerebbe piu' del setup intero.
- Mitigazione: NetworkManager con retry infinito, banda 5 GHz preferita,
  monitoring continuo del link via HA stesso.

### Perche' MX300 cloud e non EMS-ESP da subito

- EMS-ESP da' controllo locale completo ma richiede aprire la PdC e modificare hardware.
- Possibile impatto su garanzia Buderus/contratto Brusol.
- HomeCom Easy basta per il 90% del valore (boost ACS, setpoint heating).
- EMS-ESP rimane come upgrade futuro (fase 7) se la latenza cloud o le funzioni mancanti diventano bloccanti.

### Perche' niente batteria

Vincolo dato: in Belgio batterie di accumulo non sono ammesse nella configurazione del cliente. L'unica leva e' time-shift dei carichi termici (ACS + inerzia muraria via PdC).

## Modello di ottimizzazione

```
profitto_orario = pv_self_consumption_kwh * (prezzo_acquisto - prezzo_iniezione)
```

In Belgio `prezzo_acquisto` ~= 30-35 cEUR/kWh e `prezzo_iniezione` ~= 5-7 cEUR/kWh (varia per fornitore). Ogni kWh di FV consumato in casa anziche' iniettato vale ~25 cEUR. Una PdC con COP 3 che consuma 1 kWh elettrico produce 3 kWh termici: in pratica si "compra" 3 kWh di calore al costo opportunita' di 25 cEUR/3 = ~8 cEUR/kWh termico. Imbattibile con gas o resistenza elettrica.

## Storage tiering

Due livelli, gestiti separatamente:

```
SSD locale 512 GB (HOT) -- host laptop
├── Fedora root + boot                    ~30 GB
├── VM disk HA OS (qcow2)                 ~32 GB
├── Container images podman               ~20 GB
├── /home/agents/workspaces               ~100 GB (futuro)
├── Snapshot rotativi VM (ultimi 3)       ~50 GB
└── Libero / crescita                     ~280 GB

NAS Netgear via SMB (COLD) -- /mnt/nas
├── /mnt/nas/ha-backups/                  snapshot HA settimanali
├── /mnt/nas/loki-archives/               output agent worker, log run completi
├── /mnt/nas/git-mirrors/                 clone locale repo importanti (offline backup)
└── Media + file famiglia (uso esistente)
```

**Regola di scrittura**: ogni job/agente scrive **prima** su SSD locale (atomico, veloce),
poi una sync periodica (`rsync` via systemd timer ogni 15 min) sposta verso `/mnt/nas`.
Se il NAS e' offline al momento, i dati restano sull'SSD e vengono riconciliati alla
prima occasione -> **zero perdita di dati** anche se il NAS e' giu' per manutenzione.

Stessa logica per i backup HA:
- HA fa snapshot su path interno alla VM.
- Job rsync notturno (`/etc/systemd/system/ha-backup-sync.timer`) copia su `/mnt/nas/ha-backups/`.
- Retention: 7 daily + 4 weekly + 6 monthly (gestita con `rsnapshot` o `restic`).

## Sicurezza e resilienza

- Snapshot HA settimanali automatici (built-in HA).
- Snapshot VM completi via libvirt (giornalieri, retention 7 giorni).
- Configurazioni versionate in questo repo (git push frequenti).
- Segreti in `secrets.yaml` esclusi via `.gitignore`.
- DNS reservation per HomeWizard, SMA, MX300, host laptop sul router.
- BIOS host: "AC Recovery: Power On" -> auto-restart dopo blackout.
- Batteria laptop come UPS naturale per blackout < 30 min (anche se degradata).
- Tailscale per accesso remoto cifrato senza port forwarding.
- Firewall Fedora (`firewalld`) chiuso di default, apre solo: 22 (ssh interno),
  9090 (Cockpit, solo interno), 8123 (HA UI, solo interno), bridge libvirt.

## Estensioni future

| Quando | Cosa |
|---|---|
| Quando tariffa diventa dinamica | Integrazione Nordpool / fornitore -> automazioni anche su prezzo orario, non solo surplus |
| Quando serve forecast | Forecast.Solar (free tier) per anticipare le finestre di sole |
| Se garanzia PdC lo permette | EMS-ESP locale -> controllo zero-latency + diagnostiche complete |
| Se arriva EV | EVCC sopra HA -> carica solar-following |
| Se cresce il fabbisogno smart | Shelly/Zigbee plugs su lavatrice/lavastoviglie con avvio programmato |
