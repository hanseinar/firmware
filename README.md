# Meshtastic 2.7.26 med 31,25 kHz for LR11x0 (Ryfylkenett)

Gren `ryfylke/2.7.26-bw31` er offisiell Meshtastic `v2.7.26.54e0d8d` med én endring:
RadioLib hentes fra [`hanseinar/RadioLib`](https://github.com/hanseinar/RadioLib/tree/lr11x0-bw31)
(gren `lr11x0-bw31`), som er RadioLib 7.6.0 pluss støtte for 31,25 kHz båndbredde på LR11x0
(LR1110/LR1121, f.eks. Seeed T1000-E).

Opprinnelig README: [README-upstream.md](README-upstream.md).

## Hvorfor

- Offisiell RadioLib godtar bare 62,5/125/250/500 kHz for LR11x0.
- Med offisiell firmware og `lora.bandwidth 31` stopper `lora.begin()` på `setBandwidth` (feil -8).
  Frekvens, SF, CR, sync word og effekt blir aldri satt. Noden er ute av nettet og kan ikke
  reddes over mesh, bare med USB.
- Semteks egen driver (SWDR001) og LoRa Basics Modem (SWL2001) har 31,25 kHz som kode `0x02`.
  Databladene oppgir 62,5 kHz som laveste innstilling. 31,25 kHz er altså udokumentert i datablad.

SX1262-noder (f.eks. Heltec V4) støtter 31,25 kHz med offisiell firmware og trenger ikke denne grenen.

## Endringen

| Repo | Fil | Endring |
|---|---|---|
| RadioLib | `src/modules/LR11x0/LR11x0_commands.h` | Ny `RADIOLIB_LR11X0_LORA_BW_31_25 (0x02UL << 0)` |
| RadioLib | `src/modules/LR11x0/LR11x0.cpp` | `setBandwidth`: `case 15: // 31.25` |
| firmware | `platformio.ini` `[radiolib_base]` | Peker på `hanseinar/RadioLib` zip, låst til commit |

Meshtastic-koden er uendret. Firmware gjør `lora.bandwidth 31` om til 31,25 kHz.
LDRO og sendetid regnes fra `bandwidthKhz` og trenger ingen endring.

## Versjon

Versjonsteksten er `2.7.26.<commit-ID>` (fra `bin/readprops.py`). Commit-ID på denne grenen
er ulik `54e0d8d`, så hjemmelagd firmware skilles fra offisiell i `meshtastic --info`.

## Bygge

Krever Python 3 og PlatformIO.

```
git clone -b ryfylke/2.7.26-bw31 https://github.com/hanseinar/firmware
cd firmware
git submodule update --init protobufs
pip install platformio
pio run -e tracker-t1000-e
```

Resultat: `.pio/build/tracker-t1000-e/firmware-tracker-t1000-e-2.7.26.<commit-ID>.uf2`

## Flashe T1000-E (UF2)

1. Koble til USB.
2. `meshtastic --enter-dfu`
3. En USB-disk dukker opp. Kopier `.uf2`-filen dit.
4. Noden starter på nytt når kopieringen er ferdig.

UF2-flashing beholder konfigurasjon og nøkler. Ny nøkkel lages bare hvis privat nøkkel
mangler eller har lav entropi (`NodeDB.cpp`).

## Kontroll etter flashing

```
meshtastic --info
meshtastic --get security.public_key
```

- Firmware-versjon skal være `2.7.26.<commit-ID>` fra denne grenen, ikke `2.7.26.54e0d8d`.
- Offentlig nøkkel skal være uendret fra før flashing.

## Kanaler (Ryfylkenett)

| | Frekvens | BW | SF | CR |
|---|---|---|---|---|
| Kanal A | 869,628 MHz | 31,25 kHz | 6 | 4/5 |
| Kanal B | 869,580 MHz | 31,25 kHz | 6 | 4/5 |

Kanal B:

```
meshtastic --set lora.use_preset false --set lora.bandwidth 31 --set lora.spread_factor 6 --set lora.coding_rate 5 --set lora.override_frequency 869.580
```

Kanal A: samme, med `--set lora.override_frequency 869.628`.

`override_frequency` overstyrer frekvensslot (`channel_num`). Med override 0, 31,25 kHz og
slot 1 i EU_868 blir frekvensen 869,415625 MHz.

**Viktig:** Sett aldri `lora.bandwidth 31` på en LR11x0-node som kjører offisiell firmware.

## Testet

- 14. okt. 2026: T1000-E (LR1110, denne firmware) mot Heltec V4 (SX1262, offisiell),
  kanal B: direktemeldinger begge veier.
- Ikke testet: lange avstander, marginale signaler, SDR-kontroll av bredde og frekvens.
