---
title: "GitHub Pages + Jekyll"
date: 2026-05-31 13:23:27 +0200
tags: [GitHub, Jekyll]
header:
  teaser: https://github.com/user-attachments/assets/ef80f634-0b38-474d-b40f-8593aae6e1c4
comments: true
published: true
---

A régi oldal egy eléggé elavult Drupal rendszeren futott. Nehéz volt a karbantartása. Kellett valami frissebb jobb, könnyebben karbantartható.

<!--more-->

<img width="100%" alt="GitHub Pages + Jekyll" src="https://github.com/user-attachments/assets/ef80f634-0b38-474d-b40f-8593aae6e1c4" />


Szerettem a Drupalt és valamennyire értettem is hozzá. A újabb verzióra migráláshoz nem volt kedvem, mert a tárhelyen csak egyféle PHP verziót lehet használni, az újabb Drupalnak meg újabb kell. Ha átállítom friss PHP-re az összes régi oldal ami a tárhelyről megy elpusztul. És azok jellemzően olyanok, ahol nem igazán engedhető meg efféle kiesés. Meg hát időm se lett volna rá.

Mivel elég sokat dolgozom GitHubon meg Markdown fájlokkal, kézenfekvőnek tűnt, hogy a GitHub Pages + Jekyll kombinációval hozzak létre egy új oldalt. Marhára egyszerű a dolog. Minden poszt egy Markdown fájl, aminek az elején egy speciális fejléc a front matter ül. Ennek a posztnak ez a fejléce.

```yaml
---
title: "GitHub Pages + Jekyll"
date: 2026-05-31 13:23:27 +0200
tags: [GitHub, Jekyll]
header:
  teaser: https://github.com/user-attachments/assets/ef80f634-0b38-474d-b40f-8593aae6e1c4
comments: true
published: true
---
```

Ez a fájl tartalmazza a poszt címét, dátumát, címkéit, bevezetőjét, stb. De, hogy még ezen se kelljen gondolkodni az **Actions**-ben létrehoztam egy **Workflow**-t, ami a megadott paraméterek alapján létrehoz egy üres poszt vázat, amit csak át kell írni, commit, push, és pár másodperc alatt lefut a deployment és már meg is jelent a poszt.

Nem nagy varázslat, aki akarja klónozza le a [falu/falu.github.io](https://github.com/falu/falu.github.io) repót, benne van minden.
