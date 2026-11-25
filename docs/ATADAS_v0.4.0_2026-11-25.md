# Ludovika wargaming platform – átadó dokumentáció (új chathez)

Verzió: v0.4.0 · Dátum: 2026-11-25 · Készítette: Claude (az eddigi fejlesztési chat alapján)

**Éles (Vercel):** https://ludovika-wargaming-platform.vercel.app
**GitHub:** https://github.com/castor81/Ludovika-wargaming-platform (ág: `main`; a Vercel minden `main` pushra automatikusan telepít)

## 1. Cél és kontextus

A projekt a „Wargame Studio” vonaltól **független oldalág**: nagyon egyszerűsített wargaming platform a **Hőkupola 2027** Közös Közszolgálati Gyakorlathoz (NKE HHK KTVK; kb. 500 hallgató, 35 táblagép, wifi, Moodle). Három felület: COP (kivetített közös kép), Gyakorlatvezető, Csoport. Minden adatnak kiértékelhetőnek kell lennie (dashboard + export). Megjelenés: NKE.hu-arculat (arany, zöld, piros), áttetsző „glassmorph” felület, halvány NKE-logó mintás háttér.

**Megrendelő:** Dr. Varga Tamás ezredes (NKE HHK KTVK). Nyelv: magyar, tömör jelentések.
**Állandó munkaszabályok:** minden leszállított anyag verziószámmal és dátumbélyeggel (`név_vX.Y_ÉÉÉÉ-HH-NN`); minden fejlesztés után automatikus commit + push a GitHubra és újratelepítés Vercelen.

## 2. Hozzáférések és azonosítók (titkok nélkül)

| Elem | Érték |
|---|---|
| GitHub repo | `castor81/Ludovika-wargaming-platform` |
| Vercel projekt | `ludovika-wargaming-platform`, azonosító `prj_cbVyEvQPtoPd0pAoE7BeC6rybika`, csapat `team_eDSKcGFeE2zG8f7nlhYAMUOk` |
| Supabase projekt | „KM-dev”, azonosító `xaklmjaexyhcbztklyqf`, URL `https://xaklmjaexyhcbztklyqf.supabase.co` |
| Supabase kulcs | a publikus (publishable) kulcs a `js/config.js`-ben van; nem titok |
| Tároló (Storage) | nyilvános bucket: `lwp-media` |
| Gyakorlatvezetői PIN | a **hash-e** a `lwp_config` táblában (`lead_pin_sha256`, sima SHA-256 hex); a PIN-t a megrendelő ismeri, itt nincs leírva. Cseréje: Gyakorlatvezető → Beállítások, vagy SQL-ből `update lwp_config set value = to_jsonb(encode(sha256(convert_to('ÚJ-PIN','UTF8')),'hex')) where key='lead_pin_sha256'` |
| Vercel védelem | az SSO-védelem ki van kapcsolva (a hallgatók QR-linkjei miatt) |

## 3. Architektúra

- Statikus, **build nélküli** vanilla JS SPA, hash-útvonalakkal. Nincs csomagkezelő, a külső könyvtárak a `js/vendor/` alatt: supabase-js 2.45.4, Chart.js 4.4.4, qrcode-generator 1.4.4, mammoth (Word → HTML).
- Betűtípusok: Fraunces + Inter (Google Fonts). Az Optima nincs használva (licenc).
- Két háttér-adapter a `js/store.js`-ben ugyanazzal az API-val (`window.LWP`): `sbBackend()` (Supabase: realtime `postgres_changes`, 20 mp-es pótló lekérdezés, Storage-feltöltés, optimista frissítés) és `localBackend()` (`?local=1`: localStorage + BroadcastChannel, PIN: `demo`, csoportkódok: `demo-<ág>-<betű>`, pl. `demo-allam-a`).
- Globális állapot: `S` (groups, msgs, reacts, events, cfg). API: `LWP.init, send, upload, react, beat, login, setHelp, patchMsg, logEvent, setCfg, reloadEvents, status, touch, sha256, on`.

### Fájlok

| Fájl | Tartalom |
|---|---|
| `index.html` | héj, szkriptek betöltése, háttérréteg (`.bgfx`) |
| `css/style.css` | színtokenek, glass panelek, NKE-téma, QR, pop-up/sziréna animációk, reszponzív szabályok |
| `js/config.js` | `version`, `date`, Supabase URL/kulcs, `bucket`, `maxUploadMB` (50), `replyWindowMin` (30) |
| `js/store.js` | adatréteg (Supabase és helyi adapter), állapot, valós idejű frissítés |
| `js/ui.js` | segédek, hang (sziréna), értesítő (`Notif`), `Feed` (hírfolyam), `Composer` (szerkesztő), modal, lightbox, QR |
| `js/views.js` | nézetek: nyitóoldal, COP, dashboard, gyakorlatvezető, csoport, forgatókönyv-modul, útválasztó |
| `assets/` | NKE-címer, mintacsempe, forgó 3D-logó (animált WebP, 4 MB), Word-eseménysablon |
| `scenario/` | mintaforgatókönyv (17 elem, JSON) |
| `db/schema_v0.1_2026-11-23.sql` | csak leíró, megjegyzés-szintű séma (az éles DB migrációkkal készült) |

### Útvonalak

`#/` nyitóoldal · `#/cop` COP · `#/cop/dashboard` kiértékelő · `#/lead` gyakorlatvezető (PIN) · `#/g/<kód>` csoport (a QR ide mutat).

## 4. Adatmodell (Supabase, `public` séma, `lwp_` előtag)

- `lwp_groups`: `id` (pl. `allam-a`), `branch` (`allam`/`rend`/`hon`), `letter`, `name`, `token`, `last_seen`, `last_login`, `active_sec`, `help`
- `lwp_messages`: `id` uuid, `channel` (`cop` | `lead:<csoport>` | `dm:<a>~<b>`), `author_type` (`lead`/`group`/`system`), `author_id`, `kind` (`text`/`image`/`audio`/`video`/`event`/`file`), `body`, `media_url`, `media_name`, `media_type`, `pinned`, `deleted` (puha törlés), `created_at`, **`reply_to` uuid** (v0.4.0)
- `lwp_reactions`: `message_id`, `actor`, `emoji` (összetett kulcs)
- `lwp_config`: `key`, `value` jsonb: `exercise` (név, `yellow_min`, `red_min`), `lead_pin_sha256`, `scenario`
- `lwp_events`: `id`, `ts`, `group_id`, `type` (`login`/`help`/`inject`), `data`
- RLS bekapcsolva, **nyitott anonim szabályokkal** és kifejezett `GRANT`-okkal az `anon` szerepre (migrációk: `lwp_grants_anon`, `lwp_messages_reply_to`). Új tábla/oszlop felvételekor a jogosultságokat ellenőrizni kell (korábban „permission denied” hiba volt).

## 5. Funkciók (v0.4.0)

**COP (kivetítő):** 3 ág × 5 csoport (A–E) aktivitásszínnel (zöld aktív; sárga 5–15 perc; piros 15+ perc inaktív; szürke nincs belépve; küszöbök a Beállításokban); belépő QR-kódok ágak szerint csoportosítva, méretcsúszkával és kattintásos nagyítással; közös hírfolyam; dashboardra váltó gomb; hangengedélyező gomb.

**Hírfolyam:** szöveg, kép, hang (felvétel), videó, fájl (max. 50 MB); emoji-reakciók; törlés (gyakorlatvezető bármit, csoport csak a sajátját – csak UI-szintű szabály, puha törlés); „↩ Válasz” hivatkozás üzenetre (idézetsor a folyamban); új közlés először nagy pop-upban jelenik meg 2–3 mp-ig, majd beágyazódik; a gyakorlatvezető **kiemelt** közlése sziréna hanggal és pulzáló megjelenéssel jelenik meg (a böngésző csak egy felhasználói kattintás után enged hangot).

**Gyakorlatvezető:** formázható szöveg és média; COP-közlés (kiemelt jelölővel); csoport-munkaterek betekintéssel, külön chattel és külön esemény-bejátszással; forgatókönyv-modul (Word-feltöltés: az eseménysablon a repóban, `mammoth` olvassa be a táblázatot; csere/hozzáadás mód, számeltolás ütközésnél, „bejátszva” jelzés); beállítások (név, küszöbök, PIN).

**Csoport (táblagép):** három fül: COP, Gyakorlatvezető, Csoportok közötti chat; küldés/feltöltés; segélykérés jelzés; olvasatlan-számláló; aktivitásmérés (pulzus 20 mp-enként, csak ha az utolsó felhasználói bevitel ≤ 120 mp és a lap látható; az `active_sec` a ≤ 90 mp-es időközöket összegzi).

**Dashboard és export:** 6 diagram (Chart.js), csoportonkénti táblázat, KPI-k; üzenet-CSV, összesítő CSV, teljes JSON export. Az exportok **gyakorlatvezetői PIN-t** kérnek; a dashboard nézet a COP-ról PIN nélkül is nyitható. A törölt üzenetek az exportban „törölt” jelöléssel megmaradnak. Válaszidő: kifejezett `reply_to` elsőbbséggel; hivatkozás nélkül a csoport első üzenete a következő vezetői üzenetig, legfeljebb `replyWindowMin` (30) percen belül; különben a közlés „megválaszolatlan”.

## 6. Fejlesztési és telepítési folyamat

1. Klónozás: `git clone https://github.com/castor81/Ludovika-wargaming-platform`.
2. Helyi futtatás: a repo gyökerében `python3 -m http.server 8766`, majd `http://localhost:8766/index.html?local=1` (demó mód, PIN `demo`). Böngészős tesztekhez Playwright használható.
3. Verziószám: `js/config.js` (`version`, `date`) és a README fejléce + verziótörténet együtt frissítendő.
4. Commit → `git push origin main` → a Vercel automatikusan telepít (ellenőrzés: Vercel-projekt `prj_cbVyEvQPtoPd0pAoE7BeC6rybika`, állapot `READY`).
5. Adatbázis-változás: Supabase migrációval (`apply_migration`), az `anon` jogosultságok ellenőrzésével.
6. Commit-üzenet végére a munkamenet által előírt társszerzői sorok kerülnek.

## 7. Verziótörténet

- v0.4.0 (2026-11-24): válasz-hivatkozás (`reply_to`), válaszidő-időablak (hibajavítás), PIN-védett exportok
- v0.3.2 (2026-11-23): forgó, nagyobb NKE–Ludovika logó a nyitóoldalon (animált WebP)
- v0.3.1: NKE–Ludovika logó a belépő oldalon, középen
- v0.3.0: QR csoportosítás és zoom; törlés; pop-up értesítés; kiemelt közlés szirénával
- v0.2.0: forgatókönyv-események feltöltése Word-ből, eseménysablon
- v0.1.1: NKE-arculat, címermintás áttetsző háttér, NKE-logó
- v0.1.0: COP, gyakorlatvezetői és csoportfelület, dashboard

Mellékletek, amelyek nincsenek a repóban: önálló, egyfájlos HTML-változat (v0.3.2, kérésre újraépíthető: minden CSS/JS/kép beágyazva, a `supabase.js` publicPath-ellenőrzése átírva), és a felhasználói útmutató (Claude-dokumentum, v0.3.1).

## 8. Tesztelés – ismert állapot

- Helyi (`?local=1`) böngészős tesztek: popup → beágyazódás, kiemelt riasztás, törlési jogosultság, QR-modal, válasz-hivatkozás, válaszidő-ablak, PIN-kapu – mind átment.
- A megrendelő által futtatott ellenőrző teszt (15 csoport, 46 üzenet, 15 reakció, COP „15/15 aktív”, dashboard pontos) hibát egyet talált (a válaszidő-párosítás időkorlátja), ez javítva (v0.4.0).
- **Nem igazolt:** az éles Supabase-kapcsolat (valós idejű frissítés, feltöltés) a fejlesztői környezetből sosem volt valódi böngészővel végigpróbálva (csak adatbázis-szintű anonim teszt és helyi mód futott); a megrendelő tesztjének pontos környezetét nem ismerjük. A 35 táblagépes, ~500 fős valós terhelés külön nem volt kipróbálva.

## 9. Ismert korlátok és kockázatok

- **Biztonság:** a belépés csoportkód (QR) és PIN alapú UI-zár, nem erős hitelesítés; az anonim RLS nyitott, ezért a privát csevegések nem valóban titkosak, a „csoport csak a sajátját törölheti” és a PIN-es export csak a felületen érvényesül. Zárt gyakorlathoz elfogadható; élesebb használathoz Supabase Auth + szigorúbb RLS szükséges.
- **Hang:** a sziréna/hang minden eszközön egyszeri kattintást (hangengedélyező gomb) igényel; különösen a kivetítő COP-gépen.
- **Média:** 50 MB feltöltési korlát; a puha törlés miatt a törölt média a tárolóban megmarad.
- **Forgatókönyv:** a „bejátszva” jelzés az esemény számához kötött.
- **Szimulált tesztadatok:** a teszt indoklásai sablonszövegek – valós résztvevők szöveges minősége nem igazolt.

## 10. Lehetséges következő fejlesztések (javaslatok)

1. Valódi terhelésteszt (35 eszköz + COP) és az éles Supabase-kapcsolat végigpróbálása böngészőből.
2. Biztonsági megerősítés (Supabase Auth vagy aláírt csoport-tokenek, szigorúbb RLS; a dashboard opcionális PIN-védelme).
3. Gyakorlat utáni archiválás/nullázás gomb (új gyakorlat tiszta induláshoz), automatikus adattörlési ütemezés.
4. Válaszidő és aktivitás finomabb kiértékelése (pl. csoportonkénti idővonal, közlés–válasz páros nézet).
5. Moodle-integráció (belépés/átirányítás), többnyelvűség, hozzáférhetőség (kontraszt, billentyűzet).
6. Alacsonyabb sávszélességű változat a nyitóoldali logóhoz (a 4 MB-os animált WebP a táblagépek első betöltését lassítja).
