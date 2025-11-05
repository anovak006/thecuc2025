# Traefik + Docker Compose — praktični vodič

> **Zašto ovo?**  
> Traefik kao reverzni proxy objedinjuje sav HTTP/TCP/UDP promet na nekoliko ulaznih portova (entrypointa). S Docker providerom konfiguracija se piše **labelama** na kontejnerima, a Traefik ih automatski “otkriva” preko Docker API‑ja.[1](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/)[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)

## TL;DR

- **Traefik kontejner** mora imati **publishane** portove (npr. `80:80`, `443:443`) da bi bio dostupan s hosta/Interneta; samo definiranje `entryPoints` **nije** dovoljno.[3](https://doc.traefik.io/traefik/v3.3/reference/routing-configuration/tcp/middlewares/overview/)[1](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/)  
- **Backend kontejnerima ne treba `ports:`** (ne izlaži ih prema hostu). Dovoljno je da su u istoj Docker mreži s Traefikom; Traefik ide na **interni IP:port** i to je sigurnije.[4](https://drdroid.io/stack-diagnosis/traefik-udp-routing-issues)[5](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)  
- **Traefik bira port backenda** ovim redoslijedom:  
  1) ako je **exposed samo jedan port** → koristi taj; 2) ako je **više portova** → uzima **najniži**; 3) ako **nema expose‑a** ili želiš drugačiji port → postavi labelu `traefik.http.services.<ime>.loadbalancer.server.port=<PORT>`.[5](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)  
- Labele su **namespacirane**: `traefik.http.*` (HTTP), `traefik.tcp.*` (TCP), `traefik.udp.*` (UDP).[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)[6](https://dev.to/atomax/traefik-proxy-guide-configuring-public-domain-names-for-docker-containers-367m)[7](https://ioflood.com/blog/docker-compose-ports-vs-expose-explained/)

---

## 1) Pojmovi: entrypoints, routers, services, middlewares

- **EntryPoints** – portovi/protokoli na kojima Traefik sluša (npr. `:80`, `:443`). To je **statična** konfiguracija (CLI/YAML Traefika), ne Docker labele.[8](https://doc.traefik.io/traefik/v3.1/providers/docker/)  
- **Routers** – spajaju dolazne zahtjeve s *services* prema pravilima (Host/Path za HTTP; SNI/ClientIP/ALPN za TCP; UDP je specifičan).[9](https://stackoverflow.com/questions/79442789/forcing-traefik-to-listen-on-a-port-different-to-80-8080-443)[6](https://dev.to/atomax/traefik-proxy-guide-configuring-public-domain-names-for-docker-containers-367m)[7](https://ioflood.com/blog/docker-compose-ports-vs-expose-explained/)  
- **Services** – load‑balancer prema backend(ovima) (IP:port). Kod Docker providera obično navodiš samo **port**, jer IP Traefik dobije iz Docker API‑ja.[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)  
- **Middlewares** – filtri/transformacije (npr. redirect, auth, headers). Postoje HTTP i TCP middlewarei (UDP nema).[10](https://doc.traefik.io/traefik/routing/providers/service-by-label/)[11](https://community.traefik.io/t/is-the-use-of-ports-publish-or-expose-manditory-to-make-traefik-work/3310)

---

## 2) `ports` vs `expose` u Composeu

- `ports:` **objavljuje** portove kontejnera na hostu (NAT: `host:container`). Bez toga servis **nije dostupan** izvana ni na `localhost`.[3](https://doc.traefik.io/traefik/v3.3/reference/routing-configuration/tcp/middlewares/overview/)  
- `expose:` **ne** objavljuje na hostu; samo označi port(e) vidljive drugim kontejnerima u mreži (i alatima poput Traefika).[12](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-ports-and-expose-in-docker-compose)  
- Službena Compose referenca (opći pregled): [Compose file reference](https://docs.docker.com/reference/compose-file/).

**Praktična posljedica:**  
Ako Traefiku **ne** staviš `ports:` (samo `entryPoints`), **Traefik neće biti dostupan** ni izvana ni na `localhost`. Slušat će samo **unutar** Docker mreže. Objavi barem `80:80` i/ili `443:443`.[3](https://doc.traefik.io/traefik/v3.3/reference/routing-configuration/tcp/middlewares/overview/)

---

## 3) Kako Traefik bira port backend kontejnera

Traefik Docker provider radi **auto‑detekciju porta** iz Docker API‑ja:  
1) jedan exposed port → uzme ga; 2) više portova → uzme **najniži**; 3) nema expose‑a/nesuglasje → **postavi port eksplicitno** labelom `…loadbalancer.server.port`.[5](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)

> **Napomena:** backend portove **ne trebaju** biti publishani (`ports:`) — ide se preko interne mreže; to je i preporuka.[4](https://drdroid.io/stack-diagnosis/traefik-udp-routing-issues)

---

## 4) “Namespacing” labela: `traefik.http|tcp|udp`

Prefiksi određuju **protokolni sloj**:

- **`traefik.http.…`** – HTTP/L7: `routers`, `services`, `middlewares` (AddPrefix/StripPrefix, RedirectScheme, Headers, Auth, RateLimit, Retry …).[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)[10](https://doc.traefik.io/traefik/routing/providers/service-by-label/)  
- **`traefik.tcp.…`** – TCP/L4: `routers` (pravila: `HostSNI`, `ClientIP`, `ALPN`), `services`, `middlewares` (npr. `ipAllowList`, `inFlightConn`).[6](https://dev.to/atomax/traefik-proxy-guide-configuring-public-domain-names-for-docker-containers-367m)[13](https://doc.traefik.io/traefik/v2.11/providers/docker/)[11](https://community.traefik.io/t/is-the-use-of-ports-publish-or-expose-manditory-to-make-traefik-work/3310)  
- **`traefik.udp.…`** – UDP/L4: `routers` (bez pravila tipa Host/Path; praktički samo vežu entrypoint→service), `services` (lista `address: "IP:PORT"`).[7](https://ioflood.com/blog/docker-compose-ports-vs-expose-explained/)[14](https://stackoverflow.com/questions/59856722/multiple-routers-and-services-on-the-same-container-with-traefik-2)

**Sintaksa labela (obrazac):**
```
traefik.<layer>.(routers|services|middlewares).<ime>.<svojstvo>[.<pod-svojstvo>]=vrijednost
```
> `@` **nije dozvoljen** u imenima; `@docker`, `@file` i sl. koriste se samo pri **referenciranju resursa iz drugih providera** (npr. `middlewares=redirect@file`).[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)[10](https://doc.traefik.io/traefik/routing/providers/service-by-label/)

---

## 5) Mapiranje: File (YAML/TOML) → Docker labele

Traefik dinamičku konfiguraciju koju bi inače pisao kao:

```yaml
# File provider (YAML)
http:
  routers:
    my-router:
      rule: "Host(`app.example.com`)"
      service: my-svc
  services:
    my-svc:
      loadBalancer:
        servers:
          - url: "http://localhost:8080"
```

u Docker labelama pišeš ovako:

```yaml
labels:
  - "traefik.http.routers.my-router.rule=Host(`app.example.com`)"
  - "traefik.http.routers.my-router.service=my-svc"
  - "traefik.http.services.my-svc.loadbalancer.server.port=8080"  # Docker provider zna IP
```

Detaljna pravila i primjeri: [Docker routing documentation](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file).

---

## 6) Primjeri `docker-compose.yml`

### 6.1 Minimalni Traefik + jedan HTTP backend (bez publishanja backenda)

```yaml
version: "3.9"

networks:
  proxy:
    external: true   # kreiraj jednom: docker network create proxy

services:
  traefik:
    image: traefik:v3
    command:
      - --providers.docker=true
      - --providers.docker.exposedbydefault=false
      - --entrypoints.web.address=:80
      - --entrypoints.websecure.address=:443
      - --api.insecure=true           # samo za DEV! Ne u produkciji.
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"                   # dashboard (DEV)
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
    networks: [proxy]

  app:
    image: nginx:alpine
    labels:
      - traefik.enable=true
      - traefik.http.routers.app.rule=Host(`app.localhost`)
      - traefik.http.routers.app.entrypoints=web
      # nginx već EXPOSE-a 80 → port label nije nužna; inače dodaj:
      # - traefik.http.services.app.loadbalancer.server.port=8080
    networks: [proxy]
    # Bez "ports:" → backend nije objavljen prema hostu (sigurnije)
```

- Traefik je objavljen na 80/443, dashboard na 8080 (**samo DEV; ne u produkciji**).[15](https://www.reddit.com/r/Traefik/comments/137gf41/please_explain_routing_of_specific_tcpupd_port/)  
- Backend **nema `ports:`** i dostupan je isključivo kroz Traefik.[4](https://drdroid.io/stack-diagnosis/traefik-udp-routing-issues)

### 6.2 Jedan kontejner → **dva** HTTP servisa (različiti portovi)

```yaml
labels:
  - traefik.enable=true

  # Router za web (8000)
  - traefik.http.routers.web.rule=Host(`app.example.com`)
  - traefik.http.routers.web.service=svc-web
  - traefik.http.services.svc-web.loadbalancer.server.port=8000

  # Router za admin (9000)
  - traefik.http.routers.admin.rule=Host(`admin.example.com`)
  - traefik.http.routers.admin.service=svc-admin
  - traefik.http.services.svc-admin.loadbalancer.server.port=9000
```

> Kad imaš više servisa na istom kontejneru, **eksplicitno** veži router → service (`…routers.<r>.service=<s>`).[16](https://geekyants.com/en-us/blog/setting-up-traefik-proxy-with-docker-compose-a-step-by-step-guide)

### 6.3 TCP primjer (npr. PostgreSQL preko SNI, s IP allow listom)

```yaml
labels:
  - traefik.enable=true
  - traefik.tcp.routers.pg.rule=HostSNI(`db.example.com`)
  - traefik.tcp.routers.pg.entrypoints=pg
  - traefik.tcp.routers.pg.service=pg-svc
  - traefik.tcp.services.pg-svc.loadbalancer.server.port=5432
  - traefik.tcp.middlewares.pg-allow.ipallowlist.sourcerange=192.168.1.0/24,10.0.0.0/8
  - traefik.tcp.routers.pg.middlewares=pg-allow@docker
```

> TCP pravila (HostSNI/ClientIP/ALPN), TCP service i TCP middleware (`ipAllowList`).[6](https://dev.to/atomax/traefik-proxy-guide-configuring-public-domain-names-for-docker-containers-367m)[13](https://doc.traefik.io/traefik/v2.11/providers/docker/)[11](https://community.traefik.io/t/is-the-use-of-ports-publish-or-expose-manditory-to-make-traefik-work/3310)

### 6.4 UDP primjer (npr. DNS)

```yaml
labels:
  - traefik.enable=true
  - traefik.udp.routers.dns.entrypoints=dns
  - traefik.udp.routers.dns.service=dns-svc
  - traefik.udp.services.dns-svc.loadbalancer.servers=10.0.0.10:53,10.0.0.11:53
```

> UDP router u osnovi samo veže entrypoint → jedan UDP service (nema Host/Path matchera).[7](https://ioflood.com/blog/docker-compose-ports-vs-expose-explained/)[14](https://stackoverflow.com/questions/59856722/multiple-routers-and-services-on-the-same-container-with-traefik-2)

### 6.5 Globalni redirect na HTTPS iz **file providera** (referenciran iz Docker labela)

**`dynamic/middlewares.yml`**
```yaml
http:
  routers:
    http-catchall:
      rule: "hostregexp(`{host:.+}`)"
      entryPoints: ["web"]
      middlewares: ["force-https"]

  middlewares:
    force-https:
      redirectScheme:
        scheme: https
```

**Traefik servis (dio `compose.yml`)**
```yaml
volumes:
  - ./dynamic:/etc/traefik/dynamic:ro
command:
  - --providers.file.directory=/etc/traefik/dynamic
  - --providers.docker=true
```

**Primjena middlewarea iz file providera (na neki router):**
```yaml
labels:
  - traefik.http.routers.app.middlewares=force-https@file
```

> Kako se **middleware iz file providera** referencira u Docker labelama (`@file`).[17](https://doc.traefik.io/traefik/v3.3/providers/docker/)

---

## 7) Imenovanje i “scope”

- `<ime>` u `traefik.http.services.<ime>…` ili `…routers.<ime>…` je **tvoje proizvoljno ime** (drži se `[a-z0-9-]`; **bez** `@`).[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)  
- Imena vrijede unutar **istog providera** (npr. `@docker`). Ako koristiš isti alias na više mjesta, može doći do zabune; zato imenuj jedinstveno (`app-web`, `app-admin` …).[17](https://doc.traefik.io/traefik/v3.3/providers/docker/)  
- Ako **ne** specificiraš sve eksplicitno, Traefik ponekad generira imena (u dashboardu možeš vidjeti sufikse Compose projekta). Kad treba **točna referenca**, postavi imena i poveznice ručno (router → service).[18](https://stackoverflow.com/questions/66158478/how-to-set-traefik-2-4-service-name-in-docker-compose-labels)[16](https://geekyants.com/en-us/blog/setting-up-traefik-proxy-with-docker-compose-a-step-by-step-guide)

---

## 8) Sigurnosne preporuke (produkcija)

- **Ne publishaj backend kontejnere** (nema `ports:` na njima). Neka je **samo Traefik** izložen prema hostu/Internetu.[4](https://drdroid.io/stack-diagnosis/traefik-udp-routing-issues)  
- **`exposedByDefault=false`** i `traefik.enable=true` **samo** na kontenjerima koje želiš objaviti.[5](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)  
- **Docker socket** montiraj **read‑only** i budi svjestan rizika (“tko kompromitira Traefik, može do hosta”). Razmisli o **socket‑proxy** rješenju.[5](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)  
- **Dashboard**: `--api.insecure=true` koristi **samo** u DEV okruženju; u produkciji ga zaštiti autentikacijom/TLS‑om ili isključi.[15](https://www.reddit.com/r/Traefik/comments/137gf41/please_explain_routing_of_specific_tcpupd_port/)  
- **TLS**: koristi ACME/Let’s Encrypt i HSTS preko middlewarea `Headers`.[10](https://doc.traefik.io/traefik/routing/providers/service-by-label/)

---

## 9) Troubleshooting

- **“the service `<name>@docker` does not exist”**  
  Obično znači da router referencira **nepostojeći** Traefik service (ne Docker service). Provjeri da je `traefik.http.services.<ime>…` definirano na **istom** kontejneru ili da referenciraš ispravno ime/providera.[19](https://github.com/traefik/traefik/blob/master/docs/content/user-guides/docker-compose/basic-example/index.md)  
- **404/nespareni router**  
  Router nema odgovarajući `rule`/`entrypoints` **ili** nije povezan sa service‑om (`…routers.<r>.service=<s>` kad imaš više servisa).[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)  
- **Ništa nije dostupno izvana**  
  Zaboravljen `ports:` na **Traefik** kontejneru (objavi `80:80`, `443:443`).[3](https://doc.traefik.io/traefik/v3.3/reference/routing-configuration/tcp/middlewares/overview/)  
- **Backend se ne vidi**  
  Nije u istoj Docker mreži s Traefikom **ili** Traefik nema pravi port (dodaj `…loadbalancer.server.port`).[5](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)

---

## 10) Dodatni korisni linkovi

- [HTTP routing with Traefik (Docker Docs) — brzi uvod](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/)  
- [Traefik Docker routing — detaljno o labelama](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)  
- [Traefik Docker provider — port detekcija, sigurnosne napomene](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md)  
- [HTTP Routers (referenca)](https://stackoverflow.com/questions/79442789/forcing-traefik-to-listen-on-a-port-different-to-80-8080-443) · [HTTP Middlewares (pregled & popis)](https://doc.traefik.io/traefik/routing/providers/service-by-label/)  
- [TCP Routers (pravila)](https://dev.to/atomax/traefik-proxy-guide-configuring-public-domain-names-for-docker-containers-367m) · [TCP Services](https://doc.traefik.io/traefik/v2.11/providers/docker/) · [TCP Middlewares](https://community.traefik.io/t/is-the-use-of-ports-publish-or-expose-manditory-to-make-traefik-work/3310)  
- [UDP Routers](https://ioflood.com/blog/docker-compose-ports-vs-expose-explained/) · [UDP Services](https://stackoverflow.com/questions/59856722/multiple-routers-and-services-on-the-same-container-with-traefik-2)  
- [Docker Docs — Publishing vs. exposing ports](https://doc.traefik.io/traefik/v3.3/reference/routing-configuration/tcp/middlewares/overview/) · [Compose file reference (općenito)](https://docs.docker.com/reference/compose-file/)  
- [Razlika `ports` vs `expose` (objašnjenje)](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-ports-and-expose-in-docker-compose)

---

### Appendix A — Brza referenca labela (HTTP)

```text
traefik.http.routers.<R>.rule=Host(`…`) | Path(`/…`) | …
traefik.http.routers.<R>.entrypoints=web,websecure
traefik.http.routers.<R>.service=<S>
traefik.http.routers.<R>.middlewares=mw1,mw2
traefik.http.routers.<R>.tls=true
traefik.http.routers.<R>.tls.certresolver=<resolver>
traefik.http.routers.<R>.tls.domains[0].main=example.org
traefik.http.routers.<R>.tls.domains[0].sans=*.example.org

traefik.http.services.<S>.loadbalancer.server.port=8080
traefik.http.services.<S>.loadbalancer.server.scheme=http
traefik.http.services.<S>.loadbalancer.passhostheader=true
traefik.http.services.<S>.loadbalancer.healthcheck.path=/healthz

traefik.http.middlewares.<M>.redirectscheme.scheme=https
traefik.http.middlewares.<M>.headers.stsseconds=31536000
```
(Više u službenoj referenci.)[2](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)[10](https://doc.traefik.io/traefik/routing/providers/service-by-label/)

---

## Docker kontejner

Na linku https://hub.docker.com/_/traefik nalazi se docker kontejner za traefik

---

## Reference

- Docker: [Publishing & exposing ports](https://doc.traefik.io/traefik/v3.3/reference/routing-configuration/tcp/middlewares/overview/) · [HTTP routing with Traefik](https://doc.traefik.io/traefik/reference/install-configuration/providers/docker/) · [Compose file reference](https://docs.docker.com/reference/compose-file/)  
- Traefik: [Docker provider — referenca & port detekcija](https://github.com/traefik/traefik/blob/master/docs/content/middlewares/overview.md) · [Docker routing (labele)](https://stackoverflow.com/questions/77932437/traefik-how-to-configure-a-service-in-docker-compose-using-a-dynamic-yaml-file)  
- Traefik HTTP: [Routers](https://stackoverflow.com/questions/79442789/forcing-traefik-to-listen-on-a-port-different-to-80-8080-443) · [Middlewares overview](https://doc.traefik.io/traefik/routing/providers/service-by-label/)  
- Traefik TCP: [Routers — rules & priority](https://dev.to/atomax/traefik-proxy-guide-configuring-public-domain-names-for-docker-containers-367m) · [Services](https://doc.traefik.io/traefik/v2.11/providers/docker/) · [TCP Middlewares](https://community.traefik.io/t/is-the-use-of-ports-publish-or-expose-manditory-to-make-traefik-work/3310)  
- Traefik UDP: [Routers](https://ioflood.com/blog/docker-compose-ports-vs-expose-explained/) · [Services](https://stackoverflow.com/questions/59856722/multiple-routers-and-services-on-the-same-container-with-traefik-2)  
- Community / dodatno: [Ne treba publishati backend portove s Traefikom](https://drdroid.io/stack-diagnosis/traefik-udp-routing-issues) · [“Service does not exist” — troubleshooting](https://github.com/traefik/traefik/blob/master/docs/content/user-guides/docker-compose/basic-example/index.md) · [Razlika `ports` vs `expose` (objašnjenje)](https://stackoverflow.com/questions/40801772/what-is-the-difference-between-ports-and-expose-in-docker-compose)
