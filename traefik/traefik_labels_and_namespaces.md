
# Traefik Docker labele i namespaceovi: `traefik.http`, `traefik.tcp`, `traefik.udp` (kratki vodič)

Ovaj dokument sažeto objašnjava što označava prefiks `traefik.http` (i srodni `traefik.tcp`, `traefik.udp`) u Docker labelama, kako se dinamička konfiguracija preslikava iz YAML/TOML u labele te nudi kratke primjere za HTTP/TCP/UDP.

---

## 1) Statička vs. dinamička konfiguracija
- **Statička** (install) konfiguracija pokreće Traefik i definira *entrypoints*, providere itd. Primjeri (CLI):
  - `--entrypoints.web.address=:80`
  - `--providers.docker=true`
- **Dinamička** (routing) konfiguracija definira *routers*, *services* i *middlewares* — u Docker okruženju to su **labele** na vašim servisima/kontejnerima.

> Pravilo: Statičku konfiguraciju pišete kao CLI parametre ili u `traefik.yml`; dinamičku kao **Docker labele** na aplikacijskim kontejnerima.

---

## 2) Namespace (prefiks) u labelama
- `traefik.http.*`  → HTTP (L7) resursi: `routers`, `services`, `middlewares`.
- `traefik.tcp.*`   → TCP (L4) resursi: `routers`, `services`, `middlewares`.
- `traefik.udp.*`   → UDP (L4) resursi: `routers`, `services` (nema HTTP‑stil middlewarea).

### Opći obrazac labela
```
traefik.<layer>.(routers|services|middlewares).<ime>.<svojstvo>[.<pod-svojstvo>]=vrijednost
```
- `<layer>` ∈ {`http`, `tcp`, `udp`}
- `<ime>`: slobodno izabrano ime resursa (znak `@` **nije** dozvoljen). Koristi mala slova i crtice.
- Za referencu resursa iz drugog providera koristi se sufiks, npr. `…=redirect-https@file`, `…=foo@docker`.

---

## 3) Minimalni primjeri

### 3.1 HTTP (najčešći slučaj)
```yaml
labels:
  - "traefik.http.routers.web.rule=Host(`app.localhost`)"
  - "traefik.http.routers.web.entrypoints=web"
  - "traefik.http.routers.web.service=svc-web"
  - "traefik.http.services.svc-web.loadbalancer.server.port=8000"
```

### 3.2 TCP (npr. PostgreSQL preko SNI hosta)
```yaml
labels:
  - "traefik.tcp.routers.pg.rule=HostSNI(`db.example.com`)"
  - "traefik.tcp.routers.pg.entrypoints=pg"
  - "traefik.tcp.routers.pg.service=pg-svc"
  - "traefik.tcp.services.pg-svc.loadbalancer.server.port=5432"
  - "traefik.tcp.middlewares.allow.ipallowlist.sourcerange=192.168.1.0/24,10.0.0.0/8"
  - "traefik.tcp.routers.pg.middlewares=allow@docker"
```

### 3.3 UDP (npr. DNS)
```yaml
labels:
  - "traefik.udp.routers.dns.entrypoints=dns"
  - "traefik.udp.routers.dns.service=dns-svc"
  - "traefik.udp.services.dns-svc.loadbalancer.servers=10.0.0.10:53,10.0.0.11:53"
```

---

## 4) Česte napomene
- Backend kontejneri **ne trebaju** `ports:` publish kada ih poslužuje Traefik; dovoljno je da su u istoj Docker mreži i da Traefik zna **koji port** koristiti (`EXPOSE` ili labela `loadbalancer.server.port`).
- Imena resursa (router/service/middleware) drži **jedinstvenima** u okviru istog providera (`@docker`).
- Za više portova na istom kontejneru **eksplicitno** veži router → service (`traefik.http.routers.<r>.service=<s>`).

---

## 5) Reference (preporučeno čitanje)
- Docker provider (labele i port detection): https://doc.traefik.io/traefik/reference/routing-configuration/other-providers/docker/
- HTTP Routers: https://doc.traefik.io/traefik/routing/routers/
- HTTP Middlewares (pregled): https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/overview/
- TCP Routers & pravila: https://doc.traefik.io/traefik/reference/routing-configuration/tcp/router/rules-and-priority/
- TCP Middlewares (pregled): https://doc.traefik.io/traefik/reference/routing-configuration/tcp/middlewares/overview/
- UDP Routers: https://doc.traefik.io/traefik/reference/routing-configuration/udp/router/rules-priority/
- UDP Services: https://doc.traefik.io/traefik/reference/routing-configuration/udp/service/

