# THECUC 2025 – Demo infrastruktura

Ovaj repozitorij prati radni tutorial koji prati prezentaciju **“THECUC 2025 – od paketa do kontejnera”**. Cilj je pokazati kako s Docker Compose-om podići kompletan skup servisa koji demonstrira modernu kampus-infrastrukturu (reverse proxy, DNS, pošta, web, autentikacija).

## Što je uključeno

- **Traefik** (`traefik/`) – reverzni proxy, HTTPS certifikati (Let’s Encrypt / Technitium), opisana priprema mreže i .env parametara.
- **DNS** (`dns/`) – Technitium DNS server s konfiguracijom za Admin sučelje preko Traefika.
- **Mailu** (`mailu/`) – kompletan mail stack; ključni parametri u `mailu.env`.
- **FreeRADIUS** (`freeradius/`) – primjer integracije s host mrežom (radiusd s host networkingom).
- **Web** (`web/`) – Hugo + Hextra statički site, korak-po-korak upute kako generirati i servirati sadržaj.

Detaljni koraci i primjeri nalaze se u README datotekama unutar svake komponente (`traefik/README.md`, `web/README.md`, …) te u PDF-u `THECUC2025_od_paketa_do_kontejnera.pdf`.

## Brzi početak (sa tutoriala)

1. **Pripremi Docker mrežu**

   ```bash
   docker network create --driver=bridge --attachable --internal=false --subnet=172.18.2.0/24 proxy
   ```

2. **Pokreni reverzni proxy**

   ```bash
   cd traefik
   docker compose up -d
   ```

3. **Pokreni ostale servise** (prema potrebi)

   ```bash
   cd ../dns && docker compose up -d
   cd ../mailu && docker compose up -d
   cd ../freeradius && docker compose up -d
   cd ../web && docker compose up -d
   ```
4. **Pregledaj lokalni web site** – nakon dodavanja unosa u `/etc/hosts` (`127.0.0.1 thecuc.pu.carnet.hr`), otvori `https://thecuc.pu.carnet.hr/` ili prati Traefik dashboard.

## Korištenje tutoriala uz prezentaciju

- Slajdovi prate isti redoslijed: mreža → Traefik → DNS → Mailu → RADIUS → Hugo site.
- Svaka mapa u repozitoriju sadrži minimalni skup konfiguracija i skripti koje se demonstriraju tijekom predavanja.
- Za laboratorijski rad dovoljno je pratiti korake iz README datoteka i upisivati vlastite parametre (npr. API tokeni za Technitium, domene, lozinke).

Za dodatna pitanja ili proširenja koristi dokumentaciju projekata:

- [Traefik](https://doc.traefik.io/traefik/)
- [Technitium DNS](https://technitium.com/dns/)
- [Mailu](https://mailu.io/)
- [FreeRADIUS](https://wiki.freeradius.org/)
- [Hugo + Hextra](https://imfing.github.io/hextra/docs/)
