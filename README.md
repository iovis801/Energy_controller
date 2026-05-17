# Energy_controller

Configurazioni Home Assistant versionabili per ottimizzare l'auto-consumo del fotovoltaico in una casa belga senza batteria di accumulo.

## Scope del repo

Questo repo contiene **solo** i file di configurazione HA che vivono in `/config/` su una istanza Home Assistant. Non contiene HA stesso, ne' segreti, ne' lo storage runtime.

Hardware target:
- Contatore Sibelga Landis+Gyr E360-1P (porta P1-HAN)
- Inverter SMA Sunny Boy (Speedwire/Modbus locale)
- Pompa di calore Buderus MX300 (gateway K30RF, HomeCom Easy cloud)
- Lettore P1: HomeWizard Wi-Fi P1 Meter
- Host: HP Spectre x360 (i7-8705G, 20 GB RAM, 512 GB NVMe) in cantina,
  sempre alimentato, WiFi only
- OS host: Fedora Server 44
- HA gira come VM via libvirt/qemu-kvm (HA OS x86-64 image)
- Storage cold: NAS Netgero esistente, montato via SMB su `/mnt/nas`

## Struttura

```
configuration/
  helpers.yaml                          # input_number, input_boolean, input_text
  sensors/derived.yaml                  # template sensor: home_consumption, pv_surplus
  automations/
    pv_surplus_acs_boost.yaml           # boost ACS quando c'e' surplus
    pv_surplus_heating.yaml             # alza setpoint riscaldamento su surplus
    pv_surplus_notify.yaml              # notifica carichi manuali (lavatrice, etc.)
  scripts/
    restore_heatpump_state.yaml         # ripristino stato PdC post-boost
  dashboards/
    energy.yaml                         # dashboard Lovelace custom
docs/
  architecture.md                       # piano e architettura
  runbook.md                            # procedure operative
examples/
  secrets.yaml.example                  # template segreti (mai committare il vero secrets.yaml)
```

## Come si usa

1. Setup host: installa Fedora Server 44 sul laptop, abilita Cockpit, installa libvirt
   (vedi `docs/runbook.md`).
2. Crea VM HA OS via libvirt (immagine x86-64 da home-assistant.io).
3. Aggiungi le integrazioni HA: HomeWizard, SMA, HomeCom Easy via HACS.
4. Identifica gli `entity_id` reali che HA ha creato per le tue integrazioni.
5. Copia i file YAML da `configuration/` dentro `/config/` su HA (via Studio Code Server addon o git pull).
6. Sostituisci gli `entity_id` placeholder (cerca i marker `# REPLACE:`) con quelli reali.
7. In `configuration.yaml` di HA aggiungi gli include:

   ```yaml
   homeassistant:
     packages: !include_dir_named configuration
   ```
   oppure include espliciti file-per-file (vedi `docs/runbook.md`).

8. Restart HA e verifica gli acceptance test in `docs/runbook.md`.

## Marker di sostituzione

Tutti i file YAML contengono placeholder identificati da `# REPLACE: <descrizione>`. Sono i punti dove devi inserire gli `entity_id` veri della tua installazione (i nomi cambiano in base alle integrazioni che HA crea).

## License

Personale. Non distribuire.
