# trading

## Session Value Area — Rejection & Reversion (Pine Script v6)

`session_value_area_rejection.pine` — implementace zadání v3.1.

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
| 4.3 · Čítače na levelu | `Lvl.touch_l/touch_s/last_l/last_s` |
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
3. **`tolerance`** ze sekcí 5.1/5.2 zadání nečísluje — je z ní input `i_tol`
   (default **0.10 × ATR**), aby zůstalo pravidlo „všechno v ATR".
4. **Výsledky `BE` vs `TP1_only`**: `BE` = TP1 zasažen a zbytek vyhozen na
   break-even stopu; `TP1_only` = TP1 zasažen a obchod skončil timeoutem.
5. **Long i short signál na stejném baru** se ruší navzájem — protichůdný
   setup na jednom baru je artefakt, ne obchod.
6. **`skipped_concurrent`** se loguje ve formátu výsledkového alertu
   (`{cas};{SMER};{score};{trida};skipped_concurrent;;;;`), obchod se netrackuje.
7. Profil přiřazuje **celý objem baru do jednoho binu podle `hlc3`** přesně
   podle sekce 3. Nativní TradingView Volume Profile rozpouští objem přes
   rozsah baru, takže akceptační kritérium 7 (±3 ticky) je citlivé na počet
   binů — při odchylce zvyš `Počet binů`.

### Formát alertů

Vstup (sekce 9):

```
{ticker};{tf};{cas_utc};{SMER};{trida};{session};{level};{cluster_members};{cluster_score};{poradi_dotyku};{vstup_close};{vstup_next_open};{SL};{TP1};{TP2};{tp1_posunuty};{tp1_confidence};{R_body};{R_atr};{tp1_v_R};{sirka_VA_atr};{pattern}
```

`{vstup_next_open}` je ve vstupním alertu prázdné — na close signálního baru
ještě neexistuje.

Uzavření (sekce 9.3), poslední dvě pole jsou nad rámec zadání a nesou
slippage z bodu 1 výše:

```
{cas};{SMER};{cluster_score};{trida};{vysledek};{MFE_v_R};{MAE_v_R};{bary_do_TP1};{dosel_na_TP2};{vstup_next_open};{slippage_body}
```

`vysledek` ∈ `TP1_only` / `TP2` / `SL` / `BE` / `reversed` / `timeout` /
`skipped_concurrent`.

Alert v TradingView nastav na **Any alert() function call** s frekvencí
*Once per bar close*.
