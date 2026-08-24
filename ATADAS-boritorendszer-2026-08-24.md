# Átadás: blogborító-rendszer, 2026-08-24

Ez a dokumentum egy másik Claude-nak szól, aki ezt a munkát folytatja. Minden itt van,
ami a továbblépéshez kell: mi készült el, hol van, hogyan kell újragyártani, és milyen
csapdákba léptem bele, hogy te már ne lépj bele.

---

## 1. Mi készült el

**154 blogborító az új arculatban, két változatban, mindkettő él az aibrandepites.hu-n.**

| Változat | Hol jelenik meg | Méret | Mi van rajta |
|---|---|---|---|
| **Tiszta** | cikkoldal, kártya, listaoldal | 1200×800 | csak az illusztráció |
| **Címes** | `og:image`, megosztás | 1200×630 | illusztráció plusz rövid ütős cím, bal panelben |

A repóban:

```
images/blog/<slug>.jpg      a tiszta változat, átlag 84 kB
images/blog/<slug>.webp     ugyanaz webp-ben, átlag 41 kB
images/og/<slug>.jpg        a címes változat, átlag 74 kB
```

Az élő oldalon ugyanezen az útvonalon vannak. A cikkek HTML-jét **nem** írtuk át
képhivatkozás szintjén: a fájlok ugyanazon a néven cserélődtek, ezért minden
hivatkozás magától az újat kapta.

---

## 2. A generátor helye

**A kód nincs ebben a repóban.** Ott van:

```
C:\Projectek\AIBrandepites\_KOZPONT\01-brand-arculat\borito-sorozat-2026-08-24\
```

| Fájl | Szerep |
|---|---|
| `sorozat.py` | A fő lánc: cikklista → tárgykinyerés → prompt → Vertex → mérés |
| `vegleges.py` | A tiszta borító: illusztráció a keretbe illesztve, szöveg nélkül |
| `cimes.py` | A címes borító, HTML-ből renderelve |
| `utos_cim.py` | Rövid, ütős címváltozat generálása |
| `ellenorzes.py` | Teljes körű mérés minden legyártott borítóra |
| `kontaktlap.py` | Kontaktlap a szemrevételezéshez |
| `ujragyart.py` | Kijelölt borítók újragyártása, a régit `regi/`-be menti |
| `kezi-targyak.json` | Kézi felülírás, ha a szűrő túl sokat vesz el |
| `utos-cimek.json` | A 154 rövid cím, egyenként átírható |

A szabályrendszer és a promptépítés a szomszédos `borito-rendszer/` mappában:
`szabaly3.py` (jelenetvázak, stílus, tiltások) és `targy_kinyero.py` (tárgykinyerés,
klisé-szűrő).

### Újragyártás

```bash
cd _KOZPONT/01-brand-arculat/borito-sorozat-2026-08-24
python sorozat.py --mind          # hiányzó borítók legyártása, a meglévőt kihagyja
python vegleges.py                # tiszta változat
python utos_cim.py                # rövid címek, csak a hiányzókra
python cimes.py                   # címes változat
python ellenorzes.py              # teljes mérés
```

Egyetlen borító újragyártása: `python ujragyart.py <slug>`

---

## 3. Öt csapda, amibe beleléptem

### 3.1 A bal sáv nem kérhető a modelltől

A prompt kérheti, hogy a bal 36 százalék maradjon üres, de a modell **hatból négyszer
átlépte**. A megoldás szerkezeti: az illusztrációt **négyzetesen, önmagában** generáljuk,
a keretet pedig a kód építi köré. A háttérszínt magából a generált képből mintázzuk,
ezért az illesztés varrat nélküli.

### 3.2 A 429-re nem újrapróbálás kell, hanem kevesebb hívás

Címenként egy hívás 152 kérést jelentett, és a `gemini-2.5-flash`
**429 RESOURCE_EXHAUSTED**-tel válaszolt: az első teljes futásban **152-ből 132 esett ki**.
Az újrapróbálás ezt csak tolta volna. Egy hívás most **tíz címet** dolgoz fel,
strukturált JSON-nal: 152 helyett 16 kérés. Eredmény: 129/129, nulla kihagyás.

### 3.3 A tiltás megkerülhető, ha csak tárgyat tiltasz

Az első körben 13 borítón volt ítéletet mondó motívum (mérleg, serleg, érem).
Miután ezeket nevesítve tiltottam, a jelentés **túlélte és alakot váltott**: kockás
célzászló, trónszék, díjszalag. A második körben ezért a **jelentést** tiltottam:

> NOTHING in the scene may signal ranking, victory, award, honour, authority or
> superiority. If a viewer could read an object as a prize, a throne, a finish line
> or a badge of rank, leave it out.

### 3.4 A saját mérés is hazudhat

Három mérési hibám volt, mind hamis eredményt adott:

1. **A rézdetektálás tűrése 52 volt**, ami az olívazöldet (117,116,87) is rézbarnának
   vette. 30-ra szűkítve tiszta.
2. **A kétoldali pixelszám aránya rossz mutató** egyetlen kicsi, középen ülő akcentusra:
   pár pixel eltolódás 4-5-szörös arányt ad, pedig a kép helyes. Az akcentus
   **súlypontját** kell mérni.
3. **A sarok-tónus ellenőrzés elromlott a kompozit után**, mert a saját vászon sarkait
   mérte. A háttér minőségét a **nyers** képen kell mérni.

Ha egy mérés gyanúsan sokat vagy semmit nem talál, előbb a mérést ellenőrizd.

### 3.5 A képre égetett cím hiba volt

Ráraktam a teljes cikkcímet a borítóra. Rossz döntés, három okból:

- a cikkoldal **már kiírja a címet** HTML-ben, nagyban, a kép fölé
- a hosszú magyar szavak (`kisvállalkozásoknak`) **vízszintesen** kilógtak az
  illusztrációba, mert a méretezésem csak a magasságot nézte
- a kapcsolódó cikkek kártyája levágta a szöveg tetejét

Az arculat is tiltja: *„Képre égetett szöveg tilos. Minden felirat HTML-ben vagy SVG-ben."*

**A cikkoldali borítón ezért nincs szöveg.** A címes változat csak megosztásra készül,
ahol nincs mellette HTML-cím, és ott is:
- **rövid** ütős cím (max 38 karakter), nem a teljes cikkcím
- a szöveg **saját panelben** ül elválasztóval, tehát fizikailag nem tud belelógni
- a méretezés **két irányban** mér

---

## 4. Az arculati kapuk

Amit gép el tud dönteni, azt gép dönti el. `ellenorzes.py`:

- bal harmad sötét pixel: 0 kell legyen
- sarkok tónuseltérése: 6 egység fölött belső panel vagy vágóél gyanúja
- rézakcentus súlypontja összehasonlító képnél a zóna közepén
- a generált háttér papírszínű-e (nem fehér, nem sárgás)

**Végállapot: 154/154 PASS.**

Amit gép nem tud: betű, növényi alak, emberábrázolás, kitalált architektúra, ítéletet
mondó motívum. Ehhez kontaktlap kell és szemrevételezés. A 152 elemű sorozatnál ez
**33 találatot** adott 32 képen, ezek mind javítva.

---

## 5. Amit tudni kell a szerverről

**Két külön cikk-elrendezés van:**

```
/blog/<slug>/index.html        99 cikk
/blog/cikkek/<slug>.html       63 cikk
```

**Ugyanaz a cikk kétféle fájlnéven szerepelhet**, dátumelőtaggal és anélkül
(`2026-02-penz-kereses-ai` és `penz-kereses-ai`). Huszonöt helyen emiatt maradt volna
a régi kép. Ha új képet teszel ki, mindkét néven tedd ki.

**Harminc napos képcache van** (`Cache-Control: public, max-age=2592000`). Ha ugyanazon
a néven cserélsz képet, a visszatérő látogató a régit látja. Ezért verziójelölés kell:

```bash
sed -i -E 's#(images/blog/[a-z0-9-]+\.(jpg|webp))(\?v=[0-9a-z]+)?#\1?v=UJVERZIO#g' fajl.html
```

A jelenlegi verzió: `?v=20260824b`, 2840 hivatkozáson.

### Mentések a szerveren

```
~/blog-kepek-mentes-2026-08-24.tgz    a régi képek a csere előtt, 31,8 MB
~/html-mentes-2026-08-24.tgz          a HTML-ek a verziójelölés előtt
~/html-mentes2-2026-08-24.tgz         a HTML-ek az og:image átállítás előtt
~/posts-json-mentes-2026-08-24.json
```

Visszaállítás:
```bash
cd ~/domains/aibrandepites.hu/public_html/images && tar xzf ~/blog-kepek-mentes-2026-08-24.tgz
```

---

## 6. Ami hátravan

**Két törött hivatkozás**, és nem a borítómunkából származik:

```
ai-kisvallalkozasnak-10-feladat-infografika.jpg
excel-ai-automatizalas-chatgpt-infografika.jpg
```

Ezek **álló infografikák** (1080×1350) a cikk szövegében, nem borítók. A hivatkozás
megvan, a fájl soha nem került fel. Külön műfaj, külön generátor kell hozzá.

**Két meglévő infografika** a régi arculatban van (`ai-citation-...`,
`ai-email-sablonok-...`). Ha az infografika-rendszer elkészül, ezeket is cseréld.

---

## 7. Mit NE csinálj

- **Ne pushold a weboldalt ebbe a repóba.** A mappában a teljes oldal ott áll
  követetlenül. Mindig explicit `git add`-del dolgozz, soha `git add .`-tal.
- **Ne írd át a cikkek HTML-jét képhivatkozás szintjén**, ha ugyanazon a néven tudsz
  cserélni. A fájlcsere kockázatmentesebb: nincs törött markup.
- **Ne generálj borítót, ha a tárgykinyerés `needs_review`-t adott.** Inkább maradjon
  hiányzó kép, mint témailag hamis. Ilyenkor a `kezi-targyak.json`-ba írd be a tárgyakat.
- **Ne az elrendezést hibáztasd, ha a szöveg kilóg.** Az oldalsó panel jó, a teljes
  cikkcím volt hosszú.

---

## 8. Kapcsolódó memóriafájlok

A Claude-memóriában (`.claude/projects/C--Projectek-AIBrandepites/memory/`):

- `project_borito_sorozat.md` — ez a munka, részletesen
- `reference_borito_rendszer.md` — a generátor eredete és a promptrecept
- `reference_carousel_rendszer.md` — a social carousel generátor, külön műfaj
- `reference_kepi_rendszer.md` — a két képi rendszer szétválasztása

**Két külön képi rendszer van, ne keverd:** a webes (vonalas, izometrikus, fotó nélkül)
és a videós (`media-library/`, generált jelenetek, erősebb fény). Ez a munka a webes.
