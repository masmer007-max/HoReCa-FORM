# HoReCa POS Survey — Projektová dokumentace
> Aktuální stav: **funkční MVP** · Datum: 2026-05-28  
> Určeno pro: předání do Claude Code nebo jiného vývojového prostředí

---

## 1. Co projekt dělá

Anonymní jednostránkový dotazník pro stakeholdery HoReCa trhu v ČR.  
Účel: zjistit bolesti provozovatelů pro investorskou prezentaci nového AI POS systému.

### Tok uživatele
```
Otevře stránku
  ↓
  [localStorage check] — pokud již vyplnil → zobrazí "Již jste odpověděli" → KONEC
  ↓
Blok 1 — Profil provozovny
  ↓
Blok 2 — Současný POS systém
  ↓
GATE — chce pokračovat?
  ├── NE  → send(partial=true)  → Success
  └── ANO ↓
Blok 3 — Každodenní problémy
  ↓
Blok 4 — AI a funkce systému
  ↓
Blok 5 — Rozhodování a rozpočet
  ↓
send(partial=false) → Success
```

---

## 2. Struktura projektu

```
/
├── index.html        ← celá aplikace (single-file, žádný build)
└── DOKUMENTACE.md    ← tento soubor
```

**Single-file architektura** — vše (HTML + CSS + JS) je v jednom souboru.  
Žádný npm, žádný build proces, žádný framework. Otevřeš a funguje.

---

## 3. Technický stack

| Vrstva       | Technologie                                        |
|--------------|----------------------------------------------------|
| Frontend     | Vanilla HTML / CSS / JavaScript (ES2020)           |
| Databáze     | Supabase (PostgreSQL 17)                           |
| Supabase SDK | CDN: `cdn.jsdelivr.net/npm/@supabase/supabase-js@2`|
| Hosting      | GitHub Pages (plán — statický hosting zdarma)      |

---

## 4. Supabase — přístupy a konfigurace

### Credentials (již vloženy v kódu)
```
Project ID:    mfddqfpfkmxkaickpiwk
Project name:  GRID VS_masmer007
Region:        eu-north-1 (Stockholm)
DB verze:      PostgreSQL 17.6
URL:           https://mfddqfpfkmxkaickpiwk.supabase.co
Anon key:      eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJzdXBhYmFzZSIsInJlZiI6Im1mZGRxZnBma214a2FpY2twaXdrIiwicm9sZSI6ImFub24iLCJpYXQiOjE3Nzk5Mjg3NDEsImV4cCI6MjA5NTUwNDc0MX0.w4Fezb6sl5qamsGZmfn0i2S_1TIFQPMEfDIzACS12kk
```
> ⚠️ Anon key je veřejný — správně. Nikdy nevkládej `service_role` key do frontendu.

### Užitečné odkazy
- Table Editor: https://supabase.com/dashboard/project/mfddqfpfkmxkaickpiwk/editor
- SQL Editor:   https://supabase.com/dashboard/project/mfddqfpfkmxkaickpiwk/sql/new
- Auth/RLS:     https://supabase.com/dashboard/project/mfddqfpfkmxkaickpiwk/auth/policies

### Co bylo vytvořeno v Supabase (automaticky přes migrations)
1. Funkce `enable_rls_on_new_tables()` — automaticky zapíná RLS na nových tabulkách
2. Event trigger `auto_enable_rls` — volá funkci při každém `CREATE TABLE`
3. Tabulka `survey_responses` — viz schéma níže
4. RLS policy `anon can insert` — anon role může INSERT, SELECT zakázán

---

## 5. Schéma tabulky `survey_responses`

```sql
id                    uuid PRIMARY KEY DEFAULT gen_random_uuid()
created_at            timestamptz DEFAULT now()
partial               boolean DEFAULT false

-- Blok 1: Profil
q_type                text        -- restaurant | bar | hotel | catering | multi | other
q_type_custom         text        -- vlastní text pokud q_type = other
q_size                text        -- small | medium | large | other
q_size_custom         text
q_role                text        -- owner | manager | it | other
q_role_custom         text

-- Blok 2: Současný POS
q_has_pos             text        -- yes | yes_issues | no | looking | other
q_has_pos_custom      text
scale_satisfaction    integer     -- 1–5 (povinné)
q_current_pain        text[]      -- slow | expensive | no_reports | hard_to_use | no_integration | no_support | other
q_current_pain_custom text

-- Blok 3: Bolesti
q_time                text[]      -- orders | inventory | staff | reports | customers | other
q_time_custom         text
q_money               text[]      -- waste | theft | overtime | systems | errors | other
q_money_custom        text
q_one_change          text        -- volný text

-- Blok 4: AI a funkce
q_features            text[]      -- ai_orders | ai_inventory | ai_staff | ai_menu | dashboard | mobile | integrations | other
q_features_custom     text
q_ai_concern          text        -- no | privacy | staff | cost | other
q_ai_concern_custom   text

-- Blok 5: Rozhodování
q_decision            text        -- me | owner | team | other
q_decision_custom     text
q_budget              text        -- low | mid | high | enterprise | other
q_budget_custom       text
q_buy                 text        -- subscription | onetime | trial | other
q_buy_custom          text

-- Email (volitelné, ze kteréhokoli bloku)
q_email               text        -- první vyplněný email z bloků 1–5
```

---

## 6. Popis funkcí JS

### `validate(block)` — validace před přechodem
Povinná pole na každém bloku:

| Blok | Povinná pole                                      |
|------|---------------------------------------------------|
| b1   | q_type, q_size, q_role                            |
| b2   | q_has_pos, scale_satisfaction, q_current_pain (≥1)|
| b3   | q_time (≥1), q_money (≥1)                         |
| b4   | q_features (≥1), q_ai_concern                     |
| b5   | q_decision, q_budget, q_buy                       |

Při chybě: shake animace + červený label po dobu 2,5 s. Přechod se neprovede.

### `go(from, to)` — přechod mezi bloky
- Volá `validate()` pro všechny bloky kromě gate
- Skryje aktuální kartu, zobrazí cílovou
- Aktualizuje progress bar
- Scrolluje nahoru

### `collect()` — sběr dat z formuláře
- Sbírá radio (jeden výběr), checkbox (pole hodnot), scale, textarea
- Email: bere první vyplněný z `q_email_b1` až `q_email` (blok 5)
- Vrací objekt připravený pro Supabase INSERT

### `send(partial)` — odeslání
- `partial=true` → odeslání z gate (pouze bloky 1–2)
- `partial=false` → kompletní odeslání (bloky 1–5)
- Zapíše `localStorage.setItem('horeca_survey_done', '1')`
- Zobrazí Success kartu

### `scale(name, val)` — výběr na škále 1–5
- Aktualizuje `scales` objekt
- Vizuálně označí vybrané tlačítko třídou `.on`

### `tog(id, show)` / `togCb(name, wrapId)` — zobrazení "vlastní varianty"
- `tog`: pro radio — zobrazí/skryje pole po výběru "Jiné"
- `togCb`: pro checkbox — zobrazí pole pokud je zaškrtnuto "other"

---

## 7. Email pole — implementace

Email pole je na **každém bloku** (1–5) před navigačními tlačítky.
Je volitelné, odděleno horní čarou.

ID polí: `q_email_b1`, `q_email_b2`, `q_email_b3`, `q_email_b4`, `q_email` (blok 5).

`collect()` prochází pole v pořadí a bere první neprázdný email.
Do DB se uloží vždy max. jeden email v poli `q_email`.

---

## 8. Blokování opakovaného vyplnění

- Mechanismus: `localStorage` klíč `horeca_survey_done`
- Při načtení stránky: pokud klíč existuje → zobrazí karta `#balready` ("Již jste odpověděli")
- Po odeslání: `localStorage.setItem('horeca_survey_done', '1')`
- Omezení: funguje pouze ve stejném prohlížeči/zařízení
- Reset pro testování: DevTools → Application → Local Storage → smazat klíč

---

## 9. Jak spustit lokálně

### Nejjednodušší — přímé otevření
```bash
open index.html          # macOS
start index.html         # Windows
xdg-open index.html      # Linux
```
> Supabase funguje i lokálně (CDN + CORS povoleno).

### Doporučeno — lokální HTTP server
```bash
python3 -m http.server 8080
# otevři: http://localhost:8080
```

### VS Code
Rozšíření **Live Server** → pravý klik na `index.html` → "Open with Live Server"

---

## 10. Checklist testování

### Test 1 — základní průchod
- [ ] Otevřít `index.html`
- [ ] Pokusit se pokračovat bez vyplnění → shake animace, nepustí dál
- [ ] Vyplnit blok 1 (typ, kapacita, role) → přejít na blok 2
- [ ] Vyplnit blok 2 → na gate zvolit "Ne, odeslat teď"
- [ ] Zkontrolovat Supabase: nový řádek, `partial = true`

### Test 2 — kompletní průchod
- [ ] Reset localStorage (DevTools → Application → Local Storage → smazat `horeca_survey_done`)
- [ ] Projít všech 5 bloků, vyplnit vše včetně volných polí
- [ ] Na bloku 3 zadat email
- [ ] Kliknout "Odeslat dotazník"
- [ ] Zkontrolovat Supabase: `partial = false`, email uložen, všechna pole vyplněna

### Test 3 — blokování opakování
- [ ] Po odeslání obnovit stránku (F5)
- [ ] Musí se zobrazit "Již jste odpověděli" — formulář nedostupný

### Test 4 — vlastní varianta
- [ ] Na bloku 1 vybrat "Jiné" u typu provozovny
- [ ] Musí se zobrazit textové pole
- [ ] Přepnout na jinou možnost → pole zmizí
- [ ] Zkontrolovat že text se uložil v `q_type_custom`

### Test 5 — email na různých blocích
- [ ] Zadat email na bloku 1, projít až na konec bez zadání dalšího emailu
- [ ] Zkontrolovat Supabase: `q_email` obsahuje email z bloku 1

---

## 11. Nasazení na GitHub Pages (3 kroky)

### Krok 1 — Nový repozitář
1. github.com → **New repository**
2. Název: `horeca-survey`
3. Visibility: **Public**
4. **Create repository**

### Krok 2 — Nahrát soubor
1. Repozitář → **Add file** → **Upload files**
2. Přetáhnout `index.html`
3. Commit: `Initial commit` → **Commit changes**

### Krok 3 — Zapnout Pages
1. Repozitář → **Settings** → **Pages**
2. Source: **Deploy from a branch** → branch: **main** → **(root)**
3. **Save**
4. Za 1–2 min dostupné na: `https://[username].github.io/horeca-survey/`

---

## 12. Prohlížení výsledků v Supabase

### Rychlý přehled (SQL Editor)
```sql
-- Všechny odpovědi
SELECT * FROM survey_responses ORDER BY created_at DESC;

-- Počet complete vs partial
SELECT partial, COUNT(*) FROM survey_responses GROUP BY partial;

-- Rozložení typů provozoven
SELECT q_type, COUNT(*) 
FROM survey_responses 
GROUP BY q_type ORDER BY count DESC;

-- Průměrná spokojenost se současným systémem
SELECT ROUND(AVG(scale_satisfaction), 2) AS avg_satisfaction
FROM survey_responses 
WHERE scale_satisfaction IS NOT NULL;

-- Nejčastější bolesti
SELECT unnest(q_current_pain) AS pain, COUNT(*)
FROM survey_responses
WHERE q_current_pain IS NOT NULL
GROUP BY pain ORDER BY count DESC;

-- Nejžádanější AI funkce
SELECT unnest(q_features) AS feature, COUNT(*)
FROM survey_responses
WHERE q_features IS NOT NULL
GROUP BY feature ORDER BY count DESC;

-- Emaily pro beta testování
SELECT q_email, created_at 
FROM survey_responses
WHERE q_email IS NOT NULL AND q_email != ''
ORDER BY created_at DESC;

-- Volné odpovědi "jedna věc"
SELECT q_one_change, q_role, q_type, created_at
FROM survey_responses
WHERE q_one_change IS NOT NULL AND q_one_change != ''
ORDER BY created_at DESC;
```

---

## 13. TODO — co zbývá

| Priorita | Úkol |
|----------|------|
| 🔴 | Nasadit na GitHub Pages a otestovat živě |
| 🔴 | Projít checklist testování (sekce 10) |
| 🟡 | Admin stránka `admin.html` pro vizualizaci výsledků (grafy, tabulka) |
| 🟡 | Email notifikace po každém odeslání (Supabase Edge Function + Resend.com) |
| 🟢 | OG meta tagy pro sdílení odkazu na LinkedIn/WhatsApp |
| 🟢 | Sledování průchodnosti bloků (kolik lidí odpadlo na gate) |
| 🟢 | Vlastní doména místo github.io |

---

## 14. Jak pracovat s Claude Code

### Spuštění projektu
```
Mám single-file projekt index.html — anonymní dotazník pro HoReCa stakeholdery 
napojený na Supabase. Dokumentace v DOKUMENTACE.md. Chci [co chceš udělat].
```

### Příklady příkazů pro Claude Code

**Změna textu otázky:**
```
V index.html změň text otázky q_budget — přidej možnost "10 000+ Kč pro hotelové řetězce"
s hodnotou value="xlarge". Přidej ji před možnost "Jiný model".
```

**Admin panel:**
```
Vytvoř admin.html — stránku chráněnou heslem (prompt při načtení), 
která zobrazí výsledky z Supabase tabulky survey_responses jako:
1. Metrické karty (počet odpovědí, průměrná spokojenost, % partial)
2. Sloupcový graf nejčastějších bolestí
3. Tabulka posledních 20 odpovědí
Použij service_role key pouze v admin.html.
```

**Email notifikace:**
```
Vytvoř Supabase Edge Function "notify-on-response", která se spustí 
po každém INSERT do survey_responses a odešle shrnutí na email [tvůj@email.cz] 
přes Resend API (api key dostanu na resend.com).
```

**Přidání nové otázky:**
```
Do bloku 3 v index.html přidej novou otázku po q_money:
Název: "Jak řešíte aktuálně objednávky u dodavatelů?"
Typ: radio
Možnosti: Telefon / Email / Přes portál dodavatele / Obchodní zástupce / Jiné
Název pole: q_ordering
Přidej pole q_ordering text do tabulky survey_responses v Supabase.
```
