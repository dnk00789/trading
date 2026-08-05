# trading

## Session Value Area — Rejection & Reversion (Pine Script v6)

`session_value_area_rejection.pine` — implementace zadání v3.1 s patchi
v3.1a, v3.1b a v3.2a.

Cena odmítne hranu value area **dokončené** session a uzavře zpátky uvnitř →
vstup směrem do value area. TP1 = POC té samé session, TP2 = protilehlá hrana.
Levely naskládané na sobě (shluky) mají větší váhu než osamocené.

### Instalace

TradingView → Pine Editor → vlož obsah souboru → *Add to chart*.
Cílový chart timeframe je 1m–15m; nad 15m indikátor napíše varování do panelu
a nevydá signál.

### Mapa zadání → kód

| Sekce | Kde |
|---|---|
| 2 · Sessions (jen počáteční časy, nepřekrývající se) | `f_hm`, `f_sess_of`, stavový automat pod `if conf` |
| 2.1 · Studený start | `ready`, varování `čekám na data` v tabulce |
| 3 · Value area, POC jako zóna | `f_make_sess` (biny podle `hlc3`, expanze VA od POC) |
| 3.1 · Validita profilu | `f_make_sess` — min/max šířka VA v ATR, min barů, nulový objem |
| 4.1 · Váhy | `i_w_poc`, `i_w_edge` |
| 4.2 · Fixované shluky | `f_build_clusters`, volané **jen** při dokončení novější session |
| 4.3 · Čítače na levelu | `Lvl.touch_l/touch_s/last_l/last_s`, skipy `last_skip_l/last_skip_s` |
| 5.1–5.6 · Vstupní podmínky | `f_eval` |
| 6 · Stop-loss | `f_eval`, blok „6 stop-loss" |
| 7.1–7.2 · Targety a posun TP1 | `f_eval`, bloky „7.1" a „7.2" |
| 7.3 · Souběh obchodů | tracker, krok 3 (`same_busy`, `reversed`) |
| 8 · Cooldown, časové okno, min ATR % | `cd_ok`, `in_win`, `vol_ok` |
| 9 · Alerty | tracker, kroky 1–3 + `f_close_alert` |
| 9.2 · SL-first | tracker, krok 2 (`hit_sl` se vyhodnocuje jako první) |
| 10 · Vizuál | `plot`/`fill`/`box`/`label`/`table` na konci souboru |

### Rozhodnutí, která zadání nechalo otevřená

1. **Vstupní alert odchází na close signálního baru**, tedy hned když se
   rozhoduje. Open následujícího baru v tu chvíli neexistuje, takže pole
   `vstup_next_open` zůstává **prázdné** a reálná plnicí cena i slippage se
   logují až v uzavíracím alertu jako dvě pole navíc na jeho konci. Plnění
   a všechny výpočty výsledku pořád běží z open následujícího baru.
2. **`poradi_dotyku` = pořadí signálu** na daném levelu a směru od dokončení
   session, ne pořadí každého fyzického dotyku zóny. Na 1m grafu by druhá
   varianta vyčerpala limit 2 dřív, než vůbec vznikne setup.
3. **`tolerance`** ze sekcí 5.1/5.2 je input `i_tol`, default **0.25 × ATR**.
4. **`realized_R` respektuje dělenou pozici.** Obchod, který vybral TP1 a pak
   se vrátil na break-even, je `TP1_BE` s kladným `realized_R`
   (`i_tp1_pct/100 × tp1_v_R_fill`), ne nula. Ostatní: `TP2` dopočítá i zbytek
   pozice, `TP1_open` a `reversed` po TP1 berou zbytek za aktuální cenu,
   `SL` je vždy `−1.0`.
5. **Long i short signál na stejném baru** se ruší navzájem — protichůdný
   setup na jednom baru je artefakt, ne obchod.
6. **Poměry se po plnění přepočítávají z reálné plnicí ceny** (`R_fill`,
   `tp1_v_R_fill`), protože po gapu se plán a realita liší. Vstupní filtr
   `min_tp1_r` se pořád vyhodnocuje na close signálu — tam se rozhoduje —
   ale volitelný `min_tp1r_fill` (default 0 = vypnuto) umí obchod odmítnout,
   když ho plnění posune pod práh; zaloguje se jako `skipped_slippage`.
7. **Gap za stop obchod neotevře.** Když open plnicího baru leží už za SL,
   obchod v realitě nikdy nevznikl — místo něj jde do deníku `skipped_gap`.
   Dotyk levelu i cooldown se přitom **vrátí zpátky**, protože gap se pozná
   až o bar později, kdy je čítač už spotřebovaný. Vrácení se neprovede,
   pokud mezi signálem a plněním skončila session a levely se přestavěly —
   level, kterého se to týkalo, už neexistuje.
8. **`skipped_concurrent` má vlastní cooldown** (`last_skip_l`/`last_skip_s`),
   oddělený od cooldownu skutečných signálů. Bez něj by při konsolidaci na
   levelu odešla notifikace na každém baru, kde platí pattern. Počet takto
   potlačených alertů je v tabulce ve sloupci `Potl. skipy`.
9. Profil přiřazuje **celý objem baru do jednoho binu podle `hlc3`** přesně
   podle sekce 3. Nativní TradingView Volume Profile rozpouští objem přes
   rozsah baru, takže akceptační kritérium 7 (±3 ticky) je citlivé na počet
   binů — při odchylce zvyš `Počet binů`.

### Zdroj profilu

`volume` je v Pine prostě to, co vrátí feed — u CFD a FX feedů **už to tick
volume je**, žádný zvláštní režim se pro něj nepočítá. Rozlišení je proto
evidenční, ne výpočetní: tick volume měří počet změn ceny, ne velikost
obchodů, takže value area z něj může vyjít jinde a ve statistice se ty dva
světy nesmí smíchat. Režim se loguje v obou alertech spolu se symbolem.

| Režim | Kdy | Do binů |
|---|---|---|
| `vol` | burzovní feed (krypto, `ES1!`, `NQ1!`, `GC1!`) | `volume` |
| `tick` | `syminfo.type` je `cfd` nebo `forex` a `volume` je nenulové | `volume` (= tick volume) |
| `tpo` | `volume` chybí nebo je nulové ve víc než polovině barů session | `1` za bar |
| `tick_proxy` | volitelný; feed objem nedá vůbec | počet barů nižšího TF v baru |

`tick_proxy` je v defaultu vypnutý a má **výrazně kratší historii** —
`request.security_lower_tf()` má strop na počtu načtených LTF barů. Pro živé
obchodování to nevadí, pro ruční procházení historie ano. Cena do binu se
i v tomhle režimu bere z `hlc3` baru grafu.

`auto` (default) rozlišuje `vol` / `tick` / `tpo`, nikdy nesáhne po
`tick_proxy`. Když je vybraný `tick_proxy`, ale zvolený LTF není nižší než
timeframe grafu, indikátor spadne zpátky na autodetekci a napíše to do panelu.

**Doporučené symboly:** S&P 500 přes CFD brokera `US500` / `SPX500` (`tick`),
futures `ES1!` (`vol`), index `SPX` **nepoužívat** — nemá objem. Nasdaq stejně:
`US100` / `NAS100`, `NQ1!`, a `NDX` nepoužívat.

### Formát alertů

Vstup (sekce 9):

```
{ticker};{tf};{cas_utc};{SMER};{trida};{session};{level};{cluster_members};{cluster_score};{poradi_dotyku};{vstup_close};{vstup_next_open};{SL};{TP1};{TP2};{tp1_posunuty};{tp1_confidence};{R_body};{R_atr};{tp1_v_R};{sirka_VA_atr};{pattern};{profil_rezim};{symbol}
```

`{vstup_next_open}` je ve vstupním alertu prázdné — na close signálního baru
ještě neexistuje.

Uzavření (sekce 9.3), pole za `dosel_na_TP2` jsou nad rámec zadání — nesou
slippage, realizované R a zdroj dat:

```
{cas};{SMER};{cluster_score};{trida};{vysledek};{MFE_v_R};{MAE_v_R};{bary_do_TP1};{dosel_na_TP2};{vstup_next_open};{slippage_body};{realized_R};{tp1_v_R_fill};{profil_rezim};{symbol}
```

`vysledek` ∈ `SL` / `TP1_BE` / `TP2` / `TP1_open` / `timeout` / `reversed` /
`skipped_concurrent` / `skipped_gap` / `skipped_slippage`.

Všechny `skipped_*` řádky jdou ve **stejném formátu** jako uzavírací alert,
s prázdnými poli tam, kde hodnota neexistuje — sloupce v deníku tím zůstanou
zarovnané. Nová pole se proto přidávají výhradně na konec.

Alert v TradingView nastav na **Any alert() function call** s frekvencí
*Once per bar close*.
