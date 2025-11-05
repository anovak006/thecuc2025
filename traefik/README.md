# Traefik kratke upute

## Pokretanje i zaustavljanje
- Pozicioniraj se u direktorij `traefik`.
- Pokreni uslugu u pozadini: `docker compose up -d`.
- Zaustavi i ukloni kontejnere: `docker compose down`.
- Za potpuno ciscenje volumena koristi: `docker compose down -v` (opcionalno).

## Kreiranje eksterne mreze
- Izvrsi jednom prije prvog pokretanja:

```
docker network create --driver=bridge --attachable --internal=false --subnet=172.18.2.0/24 proxy
```

- Provjeri da je mreza kreirana: `docker network ls | grep proxy`.

## DNS Challenge s Technitium

Za dobivanje wildcard certifikata ili kada TLS challenge ne radi, koristi se DNS challenge s Technitium DNS serverom.

Traefik koristi Lego biblioteku za ACME protokol, što znači da podržava sve što Lego omogućava za dobivanje Let's Encrypt certifikata, uključujući DNS challenge za wildcard certifikate. Za više informacija pogledati https://go-acme.github.io/lego/dns/

Potrebne varijable okoline u datoteci `.env`:

- `TECHNITIUM_SERVER_BASE_URL`: URL do Technitium servera (npr. `http://dns-thecuc.pu.carnet.hr:5380`)
- `TECHNITIUM_API_TOKEN`: API token za pristup Technitium serveru

Kopirati env.example u .env i promijeniti potrebne podatke:
```
cp env.example .env
```

