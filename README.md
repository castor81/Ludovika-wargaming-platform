# Ludovika wargaming platform

Egyszerűsített közszolgálati wargaming platform (NKE HHK · KKGY „Hőkupola 2027” gyakorlat), glassmorph felülettel.

**Verzió:** v0.3.0 · 2026-11-23 (a „Wargame Studio” fejlesztési vonaltól független oldalág)

## Felületek
| Útvonal | Cél |
|---|---|
| `#/cop` | COP – kivetítőre szánt közös operatív kép: 3 ág × 5 csoport (A–E) aktivitásszínekkel, belépő QR-kódok, közös chat |
| `#/cop/dashboard` | Gyakorlat-kiértékelés: diagramok, összesítő táblázat, CSV/JSON export |
| `#/lead` | Gyakorlatvezető (PIN): COP-közlés, csoport-munkaterek (betekintés + külön chat/esemény), forgatókönyv-bejátszás, beállítások |
| `#/g/<kód>` | Csoportfelület (a COP-on látható QR-kód ide vezet): COP / Gyakorlatvezető / Csoportok közötti chat |

Aktivitás: zöld < 5 perc, sárga 5–15 perc, piros 15+ perc inaktivitás, szürke: nem lépett be. Az inaktivitást a csoportfelület valódi használata (érintés, kattintás, gépelés, látható lap) alapján méri a rendszer; a küszöbök a Beállításokban módosíthatók.

## Technika
Statikus, build nélküli webapp (vanilla JS) · Supabase (Postgres + Realtime + Storage; `lwp_` előtagú táblák, `lwp-media` tároló) · Vercel.
A vendorált könyvtárak: supabase-js 2.45.4, Chart.js 4.4.4, qrcode-generator 1.4.4.

`?local=1` – helyi demó üzemmód szerver nélkül (böngészőn belül, több ablakkal kipróbálható; PIN: `demo`).

## Adatbázis
`db/` – a séma és a seed. A gyakorlatvezetői PIN SHA-256 lenyomata a `lwp_config` táblában van; a Beállításokban módosítható.

## Biztonsági megjegyzés
A belépés csoportkód (QR) és PIN alapú UI-zár, nem erős hitelesítés: a táblák anonim olvasása/írása nyitott. Zárt gyakorlati használatra készült; élesebb környezethez Supabase Auth + szigorúbb RLS javasolt.

## Verziótörténet
- v0.3.0 (2026-11-23): QR-kódok ágak szerint csoportosítva, méretcsúszkával és kattintásos nagyítással; közlések törlése (gyakorlatvezető bármit, csoport csak a sajátját – puha törlés, az adat megmarad a kiértékeléshez); új közlések nagy pop-upban jelennek meg 2–3 mp-ig, majd beágyazódnak a hírfolyamba; a gyakorlatvezető kiemelt közlései sziréna hanggal és pulzáló megjelenéssel (hangengedélyező gomb).
- v0.2.0 (2026-11-23): forgatókönyv-események feltöltése Word-ből (.docx), letölthető esemény-sablon (`assets/esemeny-sablon_v0.1_2026-11-23.docx`); a forgatókönyv az adatbázisban tárolódik (`lwp_config.scenario`), törölhető, hozzáadható vagy felváltható. A meglévő „Szakmai forgatókönyv” táblázatformátuma is beolvasható.
- v0.1.1 (2026-11-23): NKE-arculat (zöld/arany, világos üveghatás), az NKE-címer ismétlődő, áttetsző háttérmintázata, NKE-logó a fejlécben és favicon; adatbázis-jogosultságok (GRANT) pótlása az anon szerepkörnek.
- v0.1.0 (2026-11-23): első működő változat – COP, gyakorlatvezetői és csoportfelület, média és hangfelvétel, emoji-reakciók, aktivitásmérés, QR-belépés, dashboard + export, forgatókönyv-bejátszás (Hőkupola 2027 minta).
