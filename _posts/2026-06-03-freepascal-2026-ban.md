---
title: "FreePascal 2026-ban?"
date: 2026-06-03 18:41:40 +0200
tags: [FreePascal, Lazarus, Programozás]
header:
  teaser: https://github.com/user-attachments/assets/b0434f70-dc1f-4d0f-8d7e-653d1a2ba6ab
comments: true
published: true
pinned: false
---

Minden évben jön egy új "ez majd mindent megold" nyelv

<!--more-->

<img width="100%" alt="Lazarus IDE" src="https://github.com/user-attachments/assets/b0434f70-dc1f-4d0f-8d7e-653d1a2ba6ab" />

Idén is tele a Hacker News meg a Reddit a Rusttal, a Pyhtonnal és a Góval. Én meg itt ülök, és egy érett, több tízezer soros desktop alkalmazást fejlesztek FreePascalban - és nem érzem, hogy lemaradtam volna bármiről. Azt is megmondom, hogy miért.

## Olyan nyelv, ami nem akar állandóan tanítani

A Pascal szintaxisa jól olvasható, explicit, és nem változik félévente. Amit öt éve írtam, az ma is fordul. Nincs "edition", nincs breaking change minden minor verzióban. A kód, amit írok, az enyém marad - nem kell utánaszaladni a nyelvnek.

## Lazarus: az IDE, amit nem kell összerakni

A Rust/Go világban a tooling sokszor "összerakós": szerkesztő + plugin + debugger, mindenki a saját ízlése szerint. A Lazarus egy komplett RAD környezet dobozból. Form designer, integrált debugger, cross-compilation - működik. Egy desktop GUI app nálam órák, nem hetek kérdése.

## Egyetlen self-contained bináris, futásidejű függőség nélkül

Lefordítom, és kapok egy natív futtathatót. Nincs runtime, nincs "telepítsd hozzá ezt a 200 MB-ot". A felhasználó letölti, elindítja, kész. Bárki, aki valaha támogatott ügyfélnél futó szoftvert, tudja, mennyit ér ez.

Oké, ebben van egy kis ferdítés, ugyanis csak Windows alatt nem függ a keretrendszertől. A Linux az más tészta. A GUI appokhoz alapból gtk2-t használ, ami őszintén szólva már nem számít modern widhetsetnek, ezért előfordulhat, hogy újabb disztrókon kissé retró megjelenésűek ezek a programok. A gtk3 támogatás ott tart, hogy úgy lett gtk4, hogy a Lazarus még a gtk3-at nem támogatja teljes körűen. Állítólag qt5 meg a qt6 szépen megy, de azt meg én nem kedvelem. A Linux desktop fragmentálsága GUI app fejlesztői szempontból megérne egy külön posztot.

## Valódi cross-platform, fájdalom nélkül

Ugyanaz a kódbázis fordul Linuxon, Windowson és macOS-en. A saját bőrömön tapasztaltam, hogy ez mennyivel kevesebb szívás, mint amire számítottam.

## Teljesítmény, amiért nem kell megküzdeni

Natívra fordul, és gyors. Nagy adathalmazok, számításigényes algoritmusok, valós idejű feldolgozás — nem a nyelv a szűk keresztmetszet. És amikor tényleg számít a sebesség vagy a memóriaigény, ott a pointer és a manuális kontroll - anélkül, hogy a borrow checker minden sorba beleszólna.

## Stabilitás mint feature

A Pascal "unalmas". Ez a legnagyobb dicséret, amit adhatok neki. Nem kell a nyelv divatját követni; lehet a problémára koncentrálni. Egy üzleti termék mögött ez az igazi érték: tíz év múlva is karbantartható lesz.

## Rust, Python, Go?

Hogy fair legyek: a Rust memóriabiztonsága zseniális rendszerszintű kódhoz, és tényleg írtam benne számításigényes programokat - élveztem. A Go a konkurens hálózati szolgáltatásokban verhetetlen. A Python pedig adatfeldolgozásra, prototípusozásra, gyors scriptelésre továbbra is utolérhetetlen. Egyik sem rossz nyelv. Csak nem minden probléma rendszerprogramozás, felhőszolgáltatás vagy egyszeri script. Egy asztali, számításigényes, hosszú életű alkalmazáshoz a FreePascal nálam ma is a racionális választás — nem nosztalgiából, hanem mérnöki döntésből.

Itt jut eszembe, miért is értékelem annyira a Pascal stabilitását. A Python 2-ről 3-ra való átállás évekig tartó fejfájás volt: a `print` utasításból függvény lett, a stringkezelés (str vs. unicode) a feje tetejére állt, az egész osztás máshogy működött, a `dict` metódusok visszatérési értéke megváltozott. Régi kódbázisok éveken át ragadtak a 2-esen, miközben a 3-as körül épült az új ökoszisztéma. Aki átélte, az tudja: egy néhány soros script átírása még megvolt, de egy nagy, élő alkalmazás migrálása komoly projekt volt - tesztekkel, `2to3`-mal, és sok káromkodással. Pont az ilyen törések miatt értékelem, hogy a Pascalban a tíz éve írt kódom ma is változtatás nélkül fordul.

---

A hype elmúlik. A kód marad. Én olyan eszközt akarok, ami tíz év múlva is ott lesz, fordul, és nem kér tőlem cserébe semmit. 2026-ban ez nekem a FreePascal + Lazarus.
