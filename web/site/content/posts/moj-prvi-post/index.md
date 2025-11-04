+++
date = '2025-11-04T17:53:58Z'
draft = true
title = 'Moj Prvi Post'
+++

# Hugo + Hextra uz Docker

Ovaj quick start slijedi vodič [Running Hugo with Caddy and Docker](https://geeklad.com/posts/running-hugo-with-caddy-and-docker/), ali umjesto KeepIt teme koristi [Hextra](https://imfing.github.io/hextra/docs/getting-started/). Primjeri pretpostavljaju da se sve pokreće iz direktorija `web/` ovog repozitorija.

## Preduvjeti

- Docker i Docker Compose v2
- Kloniran repozitorij i pripremljena eksterna mreža `proxy` (`docker network create --driver=bridge --attachable --internal=false --subnet=172.18.2.0/24 proxy`)
- Po želji, pokrenut Traefik koji reverse-proxyja uslugu (vidi `../traefik`)

## Struktura direktorija

```text
web/
├── docker-compose.yml   # Definicija Hugo kontejnera
└── site/                # Hugo site (generira se u koracima ispod)
```

## 1. Generiranje Hugo site-a

```bash
cd web
# kreira skeleton site-a u ./site
docker run --rm -v "$(pwd)/site:/src" hugomods/hugo new site .
# preuzmi vlasništvo nad generiranim datotekama ako je potrebno
sudo chown -R "$(whoami)":"$(whoami)" site
```

## 2. Dodavanje Hextra teme (git submodule)

```bash
cd web/site
git init
git submodule add https://github.com/imfing/hextra.git themes/hextra
```

Hextra primjer konfiguracije nalazi se u `themes/hextra/exampleSite`.

## 3. Postavljanje konfiguracije

Ažuriraj `site/hugo.toml` kako bi aktivirao Hextra temu i osnovne postavke:

```toml
theme = "hextra"
baseURL = "https://example.com/"
title = "My Hextra Site"
```

Preporuka je pregledati `themes/hextra/exampleSite/hugo.toml` i kopirati željene sekcije.

## 4. Dodavanje početnog sadržaja

```bash
cd web
docker run --rm -v "$(pwd)/site:/src" hugomods/hugo new content/_index.md
docker run --rm -v "$(pwd)/site:/src" hugomods/hugo new content/docs/_index.md
docker run --rm -v "$(pwd)/site:/src" hugomods/hugo new content/posts/moj-prvi-post/index.md
```

U `site/content/posts/moj-prvi-post/index.md` dodaj front matter i tekst.

## 5. Dodaj lokalno ime servisa

Za lokalno testiranje (kada želiš da `thecuc.pu.carnet.hr` pokazuje na tvoj stroj) dodaj privremeni unos u sistemsku datoteku `/etc/hosts`.

Primjer naredbi (potreban je sudo):

```bash
# Dodaj zapis (jednokratno)
echo "127.0.0.1 thecuc.pu.carnet.hr" | sudo tee -a /etc/hosts

# Alternativno: otvori datoteku u editoru i dodaj liniju ručno
sudo nano /etc/hosts
# pa dodaj: 127.0.0.1 thecuc.pu.carnet.hr
```

Kako ukloniti zapis kada više ne trebaš:

```bash
# Ukloni sve linije koje sadrže thecuc.pu.carnet.hr i napravi backup
sudo sed -i.bak '/thecuc.pu.carnet.hr/d' /etc/hosts

# (backup će biti spremljen kao /etc/hosts.bak)
```

Brzi test DNS zapisa:

```bash
dig thecuc.pu.carnet.hr
```

Važne napomene:

- Promjena u `/etc/hosts` utječe na cijeli sustav (sve aplikacije na računalu će vidjeti novu mapu).
- Ako koristiš HTTPS (Let's Encrypt / Traefik), imaj na umu da lokalni unos u `/etc/hosts` ne osigurava valjani TLS certifikat za to ime. Za lokalno testiranje HTTPS-a možeš koristiti Traefik s validnim certifikatom ili privremeno koristiti HTTP, ili ručno povjeriti samopotpisani certifikat u pregledniku.
- Nakon izmjene `/etc/hosts`, možda ćeš trebati obrisati cache preglednika ili restartati aplikaciju koja kešira DNS.

Kada završiš testiranje, obavezno ukloni unos kako ne bi ostao stalni preslik (koristi `sed` primjer iznad).


## 6. Lokalan preview

```bash
cd web
# pokreće hugo server s Hextra temom
HUGO_THEME=hextra docker compose up -d
```

Stranica je dostupna na `https://thecuc.pu.carnet.hr/`.

Za zaustavljanje:

```bash
cd web
docker compose down
```

## 6. Ažuriranje Hextra teme

```bash
cd web/site
git submodule update --remote themes/hextra
```

## 7. Primjeri dodatne konfiguracije

- U `site/config/_default/params.toml` mogu se definirati navigacija, footer, hero sekcije.
- `site/assets/` i `site/static/` koriste se za prilagodbu stilova i statičkih datoteka.

Za detaljne mogućnosti pogledaj [Hextra dokumentaciju](https://imfing.github.io/hextra/docs/guide/).

