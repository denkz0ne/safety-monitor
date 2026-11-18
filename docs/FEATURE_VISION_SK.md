# Safety Monitor – návrhy rozšírenia a produktová vízia

> **Stav: návrh / diskusia (18. 11. 2026).** Tento dokument opisuje možné **nové funkcie** forku `denkz0ne/safety-monitor`, nie existujúce schopnosti. Nie je to zoznam chýb, otvorených issues ani záväzok implementácie. Technické a bezpečnostné opravy sú vedené samostatne v [GitHub Issues](https://github.com/denkz0ne/safety-monitor/issues).

## 1. Kam má projekt smerovať

Safety Monitor má byť **centrálny dispečing bezpečnosti domu**: nielen systém, ktorý sleduje ON/OFF hlásiče a spúšťa služby HA, ale systém, ktorý vie:

- **čo sa stalo** (typ nebezpečenstva, miesto, pôvodné merania, kontext);
- **čo s čím súvisí** (senzory, sirény, ventily, miestnosti a technická infraštruktúra);
- **ako má reagovať** (pravidlá podľa udalosti, rizika, zóny a aktuálnej situácie);
- **či zásah skutočne uspel** (spätná väzba zo zariadení, nie iba odoslaný príkaz);
- **či je ochrana pripravená** (pokrytie, batérie, komunikácia, fyzické testy, funkčnosť výstupov);
- **ako bezpečne ukončiť incident** a uchovať použiteľný záznam.

Nejde o náhradu Alarmo ani o univerzálny systémový monitor. Detekcia vlámania a bežný armed/disarmed režim patria skôr do Alarmo. Ochrana pred požiarom, CO, plynom, vodou a ďalšími haváriami musí fungovať **24/7**.

### Tri oblasti ochrany

| Oblasť | Typické udalosti | Typická reakcia |
| --- | --- | --- |
| **Life Safety** | Dym, CO, plyn, nebezpečné teplo | Okamžité upozornenie a vhodné núdzové zásahy; nikdy nedeaktivovať režimom neprítomnosti |
| **Property Safety** | Únik vody, havária technológie, potenciálne škodlivé podmienky | Lokalizovať problém, obmedziť škody, kontrolovane obnoviť prevádzku |
| **Infrastructure Safety** | Nedostupné hlásiče, výpadok Zigbee/HAOS, UPS, porucha akčného člena | Oznámiť zníženú pripravenosť a koreňovú príčinu, nezahltiť používateľa následnými alarmami |

CO₂, PM2.5 a bežné environmentálne senzory môžu dopĺňať diagnostiku, **nenahrádzajú certifikované detektory dymu ani CO**. Systém nesmie byť prezentovaný ako certifikovaná požiarna signalizácia.

---

## 2. Hlavné nové moduly

### 2.1 Incident Manager – jednotná správa udalostí

**Cieľ:** Namiesto izolovaných zmien stavu vytvárať ucelené incidenty, napr. „Únik vody – technická miestnosť“.

**Navrhované schopnosti:**

- Každý incident má stabilné ID, typ, kategóriu, závažnosť, zónu, začiatok, čas poslednej zmeny a aktuálny stav.
- Združuje súvisiace senzory, merania a všetky vykonané zásahy; viacero senzorov môže potvrdzovať tú istú udalosť.
- Umožňuje **súbežné nezávislé incidenty** (voda v pivnici a dym v kuchyni) bez vzájomného prepisovania.
- Oddeľuje fyzický stav hrozby od toho, či používateľ incident videl, stíšil sirénu alebo schválil uzavretie.
- Uchováva časovú os: detekcia → klasifikácia → pokus o zásah → overenie → notifikácia → potvrdenie → obnovenie.
- Ponúka detail incidentu, históriu rozhodnutí a prepojenia na snímky kamier / záznamy meraní.

**UI:** Prehľad aktívnych incidentov s prioritou, zónou, trvaním, aktuálnou akciou a jedným kliknutím do detailu.

**Príklad:** Dva vodné senzory pod umývačkou a drezom nemusia vždy znamenať dve havárie; korelácia však musí byť vysvetliteľná, nie slepé zlučovanie podľa času.

### 2.2 Safety Playbooks – vizuálne havarijné scenáre

**Cieľ:** Konfigurovať bezpečnostné postupy bez desiatok samostatných HA automatizácií.

Pravidlo v duchu **AK → PODMIENKA → ZÁSAH → OVERENIE → ĎALŠÍ KROK**:

1. **Spúšťač:** typ udalosti, konkrétny senzor, zóna, závažnosť či kombinácia meraní.
2. **Podmienky:** prítomnosť osôb, denná doba, stav ventilu, dostupnosť výstupu, stav nadradeného systému.
3. **Zásahy:** volanie HA služby, notifikácia, siréna, svetlá, odstavenie ventilu či vybraného zariadenia.
4. **Overenie:** čakať na skutočný stav výstupu, nie iba na úspešné odoslanie príkazu.
5. **Vetvenie:** pri neúspechu upozorniť, skúsiť bezpečnú alternatívu alebo eskalovať.
6. **Obnovenie:** samostatná cesta návratu, potenciálne s povinným ručným potvrdením.

**Konkrétny playbook: únik vody pri práčke**

- Senzor nahlási vodu → incident „Únik vody“.
- Bezodkladne vydať bezpečný príkaz na uzavretie relevantného ventilu.
- Overiť polohu ventilu a podľa konfigurácie vypnúť vybrané zariadenie, pokiaľ to nepredstavuje ďalšie riziko.
- Poslať informáciu domácnosti; pri nepotvrdení / zlyhaní zásahu eskalovať.
- Neotvárať ventil automaticky len preto, že snímač vyschol.

**UI:** Blokový editor, napr. snímač → podmienka → výstup → spätná väzba → vetva úspech/neúspech. Každý blok má nastavenia v bočnom paneli, prehľadný náhľad a validáciu prepojení. Vhodné je ukladať strojovo čitateľnú definíciu playbooku, nie len graf.

### 2.3 Safety Map – mapa senzorov a pokrytia

**Cieľ:** Zobraziť, kde sa problém nachádza, a odhaliť slepé miesta pokrytia.

- Voliteľný pôdorys domu / jednoduchá miestnostná mapa.
- Vrstvy pre senzory, sirény, ventily, kamery, dôležité spotrebiče a únikové trasy.
- Farebne odlišovať stav, zníženú dostupnosť a aktuálny incident.
- Označovať zóny bez používateľom definovaného požadovaného pokrytia (napr. bez dymového hlásiča alebo senzora vody).
- Podporovať viac podlaží a jednoduché ručné umiestnenie prvkov.

**Poznámka:** Upozornenie „miestnosť bez hlásiča“ je kontrola podľa nastavenej politiky používateľa, **nie** certifikát súladu s požiarnymi normami.

### 2.4 Safety Topology – vzťahy a závislosti zariadení

**Cieľ:** Vedieť nielen *kde* je zariadenie, ale aj *čo chráni, ovláda alebo od čoho závisí*.

- Vzťahy senzor → zóna; ventil → vodná vetva; siréna → pokryté oblasti; výstup → ovládaná technológia.
- Previazanie na HA Device Registry, Entity Registry, oblasti a tam, kde to ide, na ZHA/ESPHome diagnostiku.
- Detekcia spoločnej príčiny: keď vypadne Zigbee koordinátor, nahlásiť jeden koreňový problém a súpis dotknutých hlásičov namiesto dvadsiatich rovnocenných porúch.
- Závislosti zobrazovať v diagrame aj v incidente; vysvetliť, prečo konkrétny ventil či siréna reagujú.
- Ručné modelovanie fyzických závislostí, ktoré HA metadáta samy nepoznajú (napr. konkrétna vetva vodovodu).

**Bezpečnostná hranica:** Neoverená alebo len odhadnutá závislosť nesmie viesť k automatickému vypnutiu náhodného zariadenia.

---

## 3. Ďalšie produktové funkcie

### 3.1 Preventívne varovania a kombinovanie meraní

- Rozšíriť vstupy aj na číselné `sensor` entity: teplota, vlhkosť, prietok, tlak, úroveň UPS, napájanie, prevádzkové hodnoty.
- Pravidlá s prahmi, hysterézou, trvaním a trendom (napr. rýchly rast teploty v rozvádzači).
- Senzorová fúzia: napr. prietok vody + vlhkosť + režim neprítomnosti môže spresniť situáciu.
- Odlišovať *preventívne podozrenie* od *potvrdeného akútneho nebezpečenstva*.
- Pri CO/dyme nevyžadovať ďalšie meranie ako podmienku eskalácie certifikovaného hlásiča.

### 3.2 Safety Health – pripravenosť ochranného systému

Nad rámec existujúcich upozornení na slabú batériu, offline senzory a samostatné autotesty:

- Evidencia posledného **skutočne overeného** fyzického testu detektora a výsledku.
- Vek zariadenia, odporúčaný dátum výmeny, servisné termíny a voliteľný vlastný životný cyklus.
- Dlhodobá stabilita spojenia, počet krátkych výpadkov, diagnostika Zigbee/ESPHome podľa dostupných údajov.
- Dohľad nad sirénami, ventilmi a relevantnou spätnou väzbou z akčných členov.
- Stav pripravenosti: **plné pokrytie / obmedzené pokrytie / vyžaduje zásah** s vysvetlením konkrétnych dôvodov.
- Rozlíšenie „test neprebehol“, „príkaz na test odoslaný“ a „výsledok testu potvrdený“.

### 3.3 Rodinné notifikácie a eskalácia

- Kontakty alebo skupiny príjemcov s vlastnými pravidlami pre typ udalosti a závažnosť.
- Pri Life Safety okamžite upozorniť všetkých definovaných príjemcov; pri majetkových udalostiach možnosť postupnej eskalácie.
- Akčné možnosti v mobile: **Videl som**, **Som doma**, **Volám pomoc** a podobne.
- Voliteľná eskalácia na ďalšiu osobu po čase bez potvrdenia.
- Rozlišovať **správa odoslaná**, **doručenie neoverené**, **používateľ potvrdil**, **udalosť vyriešená** – nie sú to synonymá.
- Vhodné kanály: HA Companion, štandardné `notify` služby, prípadne ďalšie nakonfigurované transporty. Nepredpokladať, že každý transport podporuje spätnú väzbu.

### 3.4 Režimy Domov / Preč / Noc / Dovolenka

**Tieto režimy nikdy nesmú vypnúť Life Safety monitoring.** Ovplyvňujú iba vhodný spôsob reakcie:

- **Domov:** lokálne hlasové oznámenie, svetlá, prioritná informácia prítomným.
- **Preč:** agresívnejšie protihavarijné opatrenia tam, kde sú bezpečné (napr. vodný ventil), vzdialené notifikácie.
- **Noc:** výraznejšie núdzové osvetlenie a výstraha; prípadne osvetlenie únikovej trasy.
- **Dovolenka:** upravené prahy majetkových udalostí a eskalácia na poverený kontakt.

Režim sa dá odvodiť z HA `person`, prítomnosti, času a vstupov, s voliteľným manuálnym nastavením.

### 3.5 Incident Recovery – kontrolovaný návrat do prevádzky

- Po incidente ponúknuť kontrolný zoznam podľa typu udalosti.
- Pri vode: senzor je suchý, ventil je stále zatvorený, kontrola miesta prebehla, otvorenie vody vyžaduje potvrdenie.
- Pri technickej havárii: zariadenie zostáva odstavené, kým nie je známa bezpečná podmienka obnovenia.
- Podporiť povinný ručný krok, časové obmedzenia a možnosť „neobnovovať automaticky“.
- Oddeliť zrušenie akustickej výstrahy od fyzického uzavretia incidentu a obnovy prevádzky.

### 3.6 Evidence Capture – dôkazy a report incidentu

- Uložiť časovú os udalosti, kľúčové stavy a merania pred/po vzniku.
- Voliteľne priložiť snapshot z relevantnej kamery cez HA; nie automaticky z každého kamerového zdroja.
- Uchovať výsledok jednotlivých zásahov vrátane potvrdenej/nezistenej polohy výstupu.
- Export incidentu do ľahko čitateľného reportu (napr. JSON/HTML/CSV) pre diagnostiku alebo poisťovňu.
- Nastaviteľná retencia, prístupové oprávnenia a rešpektovanie súkromia domácnosti.

### 3.7 Safety Watchdog – nezávislý dohľad nad HA

- Voliteľný **externý** ESPHome alebo iný samostatný uzol kontroluje dostupnosť HAOS/Safety Monitor.
- Pri výpadku centrálneho systému môže nezávisle spustiť **vopred definovanú lokálnu** výstrahu.
- Ak má upozorňovať vzdialene, musí mať vlastný komunikačný kanál nezávislý od HA (samotné odoslanie udalosti do HA pri jeho výpadku nestačí).
- Rozlíšiť výpadok internetu, HA, MQTT, Zigbee a samostatného uzla podľa toho, čo sa dá spoľahlivo zistiť.
- Lokálne autonómne certifikované hlásiče fungujú nezávisle; watchdog ich nenahrádza.

---

## 4. Ukážky správania

### Scenár A – únik vody

1. Senzor pod práčkou sa aktivuje; Incident Manager vytvorí jeden incident.
2. Playbook zavrie ventil príslušnej vetvy a overí jeho skutočný stav.
3. Na mobile sa objaví presná zóna, senzor, stav ventilu a tlačidlo potvrdenia.
4. Ak ventil nereaguje, spustí sa bezpečná eskalácia a incident sa **neoznačí za úspešne zabezpečený**.
5. Recovery po kontrole miesta žiada explicitné schválenie opätovného otvorenia.

### Scenár B – výpadok Zigbee koordinátora

1. Safety Topology identifikuje stratu nadradenej komunikačnej súčasti.
2. Safety Health označí dotknuté senzory ako neoveriteľné a zníži úroveň pripravenosti.
3. Vytvorí sa jeden hlavný infraštruktúrny incident s výpisom závislých zariadení.
4. Používateľ dostane konkrétnu informáciu o rozsahu výpadku, nie desiatky duplicitných alarmov.
5. Samostatné fyzické hlásiče naďalej fungujú autonómne, pokiaľ to podporuje ich konštrukcia.

### Scenár C – noc a dym

1. Dymový hlásič okamžite aktivuje Life Safety incident bez čakania na ďalší senzor.
2. Playbook spustí nakonfigurované lokálne výstrahy, upozornenia a vhodné opatrenia.
3. Režim Noc môže doplniť osvetlenie únikovej trasy; nesmie znížiť závažnosť ani potlačiť hlásenie.
4. Detail incidentu zobrazuje priebeh a spätnú väzbu; ticho sirény neznamená, že nebezpečenstvo pominulo.

---

## 5. Odporúčané poradie rozvoja

| Etapa | Rozsah | Prečo takto |
| --- | --- | --- |
| **A – Základ doménového modelu** | Incident Manager, typy nebezpečenstiev, incident IDs, zóny, história, API | Všetko ostatné potrebuje jednotný model incidentov |
| **B – Riadenie reakcií** | Safety Playbooks, spätná väzba z výstupov, Recovery, rodinné eskalácie | Najväčší praktický prínos – reakcia je zrozumiteľná a kontrolovateľná |
| **C – Diagnostika a súvislosti** | Safety Health, Topology, preventívne merania, Watchdog | Systém vie rozlíšiť ochranu v poriadku vs. stratu pokrytia |
| **D – Vizualizácia a dôkazy** | Safety Map, Evidence Capture, detailnejšie UX, pokročilé režimy | Lepšie ovládanie a vyhodnotenie reálnych udalostí |

**Tri priority:** (1) Incident Manager, (2) Safety Playbooks, (3) Safety Health + Topology.

Etapy sú produktový návrh, nie termíny. Pred pripojením reálnych sirén, vypínaním technológií a vodnými ventilmi musia byť vyriešené príslušné kritické bezpečnostné chyby existujúcej implementácie. Nie sú predmetom tohto dokumentu; pozri samostatné issues.

## 6. Čo zámerne nie je cieľ

- **Nie druhé Alarmo:** nebudovať duplicitu systému pre vlámanie a ozbrojovanie domu.
- **Nie druhý systémový log:** nezahlcovať Safety Monitor každým problémom NAS-u, Wi-Fi alebo ľubovoľnej integrácie; sledovať iba technické udalosti, ktoré sú relevantné pre bezpečnosť alebo pripravenosť ochrany.
- **Nie plošné vypínanie bezpečnosti:** Domov / Preč / Noc / Dovolenka menia reakcie, nie 24/7 dohľad.
- **Nie predstieranie potvrdenia:** odoslaný príkaz nie je fyzicky potvrdený zásah; poslaná push správa nie je potvrdené prečítanie.
- **Nie nebezpečné automatické obnovenie:** po havárii sa ventily, elektrické rozvody a technológia nemajú obnovovať len na základe zmiznutia jedného alarmu.

## 7. Princípy pre návrh nových modulov

1. **Bezpečnosť pred komfortom:** typy nebezpečenstva majú jasne definovanú závažnosť a fail-safe správanie.
2. **Vysvetliteľnosť:** pri každej automatickej reakcii možno spätne zistiť, *prečo* nastala.
3. **Overiteľné účinky:** rozlišovať odoslanie služby, stav zariadenia a skutočný výsledok.
4. **Viac súbežných incidentov:** žiadny jeden globálny stav nesmie zneviditeľniť inú haváriu.
5. **Bezpečný testovací režim:** simulácie majú používať oddelený vstup a virtuálny transport akcií, nie vypnutie ochrany celého domu.
6. **Postupné rozšírenie:** nové moduly majú fungovať aj bez kamier, pôdorysu, UPS či Zigbee; závislosti sú voliteľné.
7. **Súkromie:** prístup k incidentom, kamerám, histórii a kontaktom riadiť oprávneniami a retenciou.
8. **HA-native prístup:** využiť registre, oblasti, udalosti, služby a existujúce entity Home Assistanta; neklonovať ich tam, kde to nedáva zmysel.

---

*Pripravené pre komunitný fork [denkz0ne/safety-monitor](https://github.com/denkz0ne/safety-monitor). Zoznam otvorených defektov a opráv je samostatne v Issues. Tento dokument ostáva mimo issue tracker.*
