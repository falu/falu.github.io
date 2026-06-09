---
title: "Házi Spotify"
date: 2026-06-09 22:34:38 +0200
tags: [Zene, Linux, Android, Docker, Navidrome, Tailscale, Subsonic, Last.fm, NAS]
header:
  teaser: https://github.com/user-attachments/assets/a2082783-dce0-459b-ace7-1a4115b98e03
comments: true
published: true
pinned: false
---

Egy történet arról, hogyan lett a beragadt Spotify-rádióból saját, önállóan üzemeltetett zenei szerver --- avagy hogyan vettem vissza az irányítást a zenehallgatás felett?

<!--more-->

<img width="100%" alt="image" src="https://github.com/user-attachments/assets/a2082783-dce0-459b-ace7-1a4115b98e03" />


## Ahol minden elkezdődött: a beragadt rádió

Mostanában sokat autózom, és vezetés közben kényelmes megoldás kellett a zenehallgatásra. Évekig a Spotify volt erre a kézenfekvő válasz elindítok pl. egy Tool rádiót, és megy magától. Csakhogy egy idő után feltűnt, hogy a "rádió" nem igazán rádió: nagyjából ugyanazt az ötven számot pörgeti véletlen sorrendben, újdonság alig kerül be. Ja és tök mindegy, hogy Tool vagy Metallica rádiót indítok, ugyanazok vannak mindkettőben. A Daily Mix és a Discover Weekly meg tele van olyan zenekarokkal, amiknek a felét egy perc után lekapcsolom.

Az igazság az, hogy a Spotify algoritmusa a hallgatási szokásaidhoz idomul, és ha mindig ugyanúgy hallgatsz, egyre szűkebb körben forog. Egy idő után rájöttem, hogy ez engem zavar és nem akarom, hogy egy algoritmus döntse el helyettem, mit hallgatok, és főleg nem akarok ezért havidíjat fizetni.

Régen, amikor kevesebbet utaztam, munka közben a saját NAS-omról megosztott hallgattam a zenét, desktop klienssel. Az volt az igazi: a saját gyűjteményem, a saját ízlésem, semmi ajánlóalgoritmus. Felmerült a kérdés: miért ne lehetne ezt mobilon is, autóban is?

## A terv: saját zenei szerver a NAS-on

A célom egyszerű volt:

- a saját zenei gyűjteményem elérése mobilról, bárhonnan, Android Auton keresztül az autóban,
- last.fm scrobble, mert a statisztikát szeretem,
- és mindezt biztonságosan, anélkül hogy bármit kinyitnék a routeren a netre.

A gyári NAS-appok (DS audio és társaik) ezt csak félig-meddig tudják, és elég kezdetlegesek. A self-hosted streaming szerver + rendes kliens kombó viszont sokkal jobb élményt ad.

A hardver adott volt: egy **Asustor AS4004T** (ARM-alapú, 2 GB RAM). Nem egy erőgép, de a választott szerver elenyésző erőforrást igényel.

## A három építőelem

### Navidrome – a szerver

A [Navidrome](https://www.navidrome.org/) ma gyakorlatilag a de facto választás a self-hosted zenei streamelésre. Go-ban íródott, könnyű, gyors, Docker-image formájában fut bármilyen NAS-on, és ami a lényeg: **Subsonic API-kompatibilis**, ezért rengeteg jó mobilkliens létezik hozzá. Beépített last.fm (és ListenBrainz) scrobble támogatással is jön.

Mivel az SSH-t biztonsági okokból letiltottam a NAS-on (egy korábbi szerver-incidens után ez nálam alapelv lett), a teljes telepítést grafikusan, **Portainer**-rel oldottam meg. Az App Centralból feltettem a Docker Engine-t és a Portainer CE-t, majd a Portainer **Stacks** funkciójával egy docker-compose-t illesztettem be:

```yaml
services:
  navidrome:
    image: deluan/navidrome:latest
    container_name: navidrome
    restart: unless-stopped
    ports:
      - "4533:4533"
    environment:
      ND_SCANSCHEDULE: 1h
      ND_LOGLEVEL: info
      ND_SESSIONTIMEOUT: 24h
    volumes:
      - /volume1/Docker/navidrome/data:/data
      - /volume1/Music:/music:ro
```

Két dologra érdemes figyelni: a zenei mappa útvonalát a NAS tényleges megosztásneveihez kell igazítani (a File Explorerben ellenőrizhető), a `:ro` pedig azt jelenti, hogy a Navidrome csak olvassa a zenét – ez biztonsági szempontból jó, érdemes rajta hagyni.

Deploy után a böngészőből a `http://<NAS-IP>:4533` címen jött a kezdő admin-fiók beállítása, majd a szerver szépen elkezdte beszkennelni a gyűjteményemet. Az ARM CPU-n az első scan eltart egy darabig, de utána már csak inkrementálisan dolgozik.

### Tailscale – a biztonságos elérés

Hogy autóban, mobilneten is elérjem a NAS-t, kellett valamilyen kapcsolat kívülről. A klasszikus út a DDNS + portforward lenne, de ezt szándékosan kerültem: nem akarok portot nyitni a netre. Helyette a **[Tailscale](https://tailscale.com/)** jött, ami egy WireGuard-alapú mesh VPN.

A lényege, hogy a gépeid (a NAS és a telefon) egy privát hálózatba (tailnet) kerülnek, mindegyik kap egy fix `100.x.y.z` címet, és úgy érik el egymást, mintha egy LAN-on lennének **portnyitás nélkül**. Az Asustornál van rá natív App Central-os csomag, szóval szintén grafikusan, terminál nélkül telepíthető.

Egy fontos tanulság a setup közben: az első indítás után a NAS Tailscale-kliense egyszer kiesett a kapcsolatból (az admin felületen szürke pötty jelezte), és emiatt nem tudtam csatlakozni. Újraindítás és újbóli bejelentkezés után rendben volt. Hogy ez ne ismétlődjön hónapok múlva, a Tailscale admin felületén (Machines → a NAS sora → `...` → **Disable key expiry**) kikapcsoltam a kulcs lejáratát a NAS-ra. Így nem fog magától "kiesni" egy idő után.

### Symfonium – a kliens

A kliens oldalon a **[Symfonium](https://symfonium.app/)** (Android, fizetős, pár euró) lett a befutó, és bőven megérte. Subsonic-kompatibilis, így problémamentesen csatlakozik a Navidrome-hoz. A beállítás: a szerver típusa Subsonic, a cím a NAS Tailscale-IP-je (`http://100.x.y.z:4533`), majd a Navidrome-ban létrehozott felhasználónév és jelszó.

Amit a Symfonium tud, és amiért a gyári app meg sem közelíti:

- **Android Auto** támogatás – ez volt a kulcs, hiszen a Spotify-t is így hallgattam a kocsiban.
- **Gapless lejátszás, ReplayGain**, és minden, amit egy igényes lejátszótól elvársz.
- **Offline letöltés** is van benne, ha mégis kellene – nálam ritkán fordul elő, hogy ne lenne net az úton, úgyhogy alapból streamelek a NAS-ról, de jó tudni, hogy a lehetőség adott.

## A last.fm-kérdés: itt jött a Pano Scrobbler

A scrobble volt az utolsó hiányzó láncszem, és itt akadtam el. Elvileg a Symfonium tud last.fm scrobble-t, de nem tudtam beállítani.

A megoldás végül egy **dedikált scrobbler-app**: a **[Pano Scrobbler](https://github.com/kawaiiDango/pano-scrobbler)** lett. Ez nem a Symfonium belső logikájára támaszkodik, hanem az Android **media-session** alapján figyeli, mi szól éppen, és önállóan küldi a scrobble-t a last.fm-re. Mivel a Symfonium media-sessiont használ (ez alapból be van kapcsolva), a Pano gond nélkül "kihallja" belőle a lejátszást.

A beállítás egyszerű: telepíted a Pano Scrobblert, bejelentkeztetek a last.fm-fiókoddal, az alkalmazáslistában engedélyezed a Symfoniumot, és onnantól ami a Symfoniumban szól, az felkerül a last.fm-re. Fontos: ha a Pano-t használod scrobblerként, a Symfonium saját last.fm-összekötését hagyd kikapcsolva, különben dupla scrobble lenne belőle. Streamelésnél ez stabilan működik, és az első időkben azért érdemes ránézni a profilodra, hogy tényleg minden felkerül-e.

## Az eredmény

A teljes lánc tehát:

**saját NAS → Navidrome → Tailscale → Symfonium → Android Auto → Pano Scrobbler → last.fm**

Pontosan az az élmény, ami a desktop lejátszókkal megvolt/megvan munka közben, csak most mobilon, az autóban is Android Auton keresztül, bárhonnan. Ahogy korábban a Spotify-t is ezen át hallgattam, csak most a saját gyűjteményemet streamelem a NAS-omról – a saját ízlésem, semmi ajánlóalgoritmus, semmi havidíj –, és mindezt biztonságosan, a netre nyitott port nélkül.

Pár dolog, amit érdemes az elején elintézni a tartós nyugalomért: a Tailscale key expiry kikapcsolása a NAS-on, az automatikus indítás ellenőrzése (hogy újraindítás után magától visszajöjjön), a NAS energiagazdálkodásának áttekintése (hogy ne aludjon el a szerver), és persze egy rendes, erős jelszó a teszteléshez használt ideiglenes helyett.

![A zenei stack architektúrája](/assets/images/zenei-stack-architektura.svg)

Az első beállítás kicsit döcögős volt – ez a self-hosting velejárója –, de utána ez egy "beállítom és elfelejtem" rendszer lett. 

🤘
