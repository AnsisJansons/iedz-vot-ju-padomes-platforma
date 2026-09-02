# Limbažu novada iedzīvotāju padomes platforma

Palaižams prototips iedzīvotāju jautājumu reģistrēšanai un iesnieguma sagatavošanai. Tas īsteno jautājuma ievadi, lietas dzīves cikla modeli, meklēšanas moduļa robežu, versētu iesnieguma projektu, skaidru apstiprināšanu, PDF ģenerēšanu un nomaināmu elektroniskās parakstīšanas provider slāni. Meklēšana/AI pēc noklusējuma ir izslēgti, tādēļ prototips neizdomā atbildes vai avotus.

## Prasības un palaišana

- Node.js 22.5 vai jaunāks (tiek izmantots iebūvētais `node:sqlite`)
- Nokopē `.env.example` kā `.env` un norādi īsto `MUNICIPALITY_EMAIL` tikai pirms e-pasta moduļa ieviešanas.
- Palaid `npm start` (Windows vidē ar bloķētiem PowerShell skriptiem: `npm.cmd start`).
- Atver `http://localhost:3000`.
- Testi: `npm.cmd test`.

Serveris pirmajā palaišanas reizē izveido `data/platform.db` un `uploads/`. Pielikumi netiek glabāti publiskajā mapē. Prototips neietver autentifikāciju, pretvīrusu pārbaudi un reālu eParaksta/e-pasta integrāciju, tāpēc nav paredzēts produkcijai.

## Moduļi

- `public/` — pieejama iedzīvotāja forma;
- `src/api/` — HTTP API un validācija;
- `src/db/` — SQLite savienojums un sākotnējā shēma;
- `src/cases/` — lietas ID, saglabāšana un statusi;
- `src/audit/` — nemaināms būtisko darbību žurnāls;
- `src/search/` — avotu prioritātes un nomaināma meklēšanas saskarne;
- `src/signing/` — parakstīšanas sesijas, provider līgums, izstrādes mock un eParaksts PortalSign adaptera robeža;
- `src/delivery/` — nosūtīšanas serviss, SMTP un mock e-pasta provideri, idempotence un kontrolēts retry;
- `src/responses/` — ienākošo pašvaldības atbilžu droša sasaistīšana un glabāšana;
- `src/cabinet/` — personīgā kabineta lietu apkopojums un notikumu laika līnija;
- `src/admin/` — administratora pārskati un filtrēta lietu pārvaldība;
- `src/modules/` — turpmāko AI, dokumentu, eParaksta, e-pasta, atbilžu un paziņojumu moduļu robežas;
- `docs/architecture.md` — arhitektūra un nākamie soļi.

## API

- `POST /api/cases` — `multipart/form-data`, izveido lietu un drošu piekļuves saiti;
- `GET /api/cases/:caseNumber` — lietas publiski drošais kopsavilkums;
- `GET|POST /api/cases/:caseNumber/application` — apskata, ģenerē vai versē projektu (Bearer marķieris);
- `POST /api/cases/:caseNumber/application/approve` — apstiprina priekšskatīto versiju;
- `POST /api/cases/:caseNumber/application/pdf` — ģenerē PDF tikai no apstiprinātās versijas;
- `POST /api/cases/:caseNumber/application/signing` — pēc lietotāja darbības sāk parakstīšanas sesiju;
- `GET|POST /api/signing/sessions/:id` — sesijas statuss, pabeigšana vai atcelšana ar atsevišķu sesijas marķieri;
- `POST /api/inbound/municipality` — servera-to-servera webhook pašvaldības atbildei (`INBOUND_WEBHOOK_SECRET`);
- `GET /api/admin/response-overdue` — termiņu pārskats administratoram (`ADMIN_API_TOKEN`);
- `GET /api/cabinet` — lietotāja lietas ar drošu kabineta marķieri;
- `GET /api/admin/cases` — filtrēts administratora lietu pārskats;
- `POST /api/admin/cases/:caseNumber/retry-delivery` — viens kontrolēts nosūtīšanas retry administratoram;
- `GET /api/health` — veselības pārbaude.

Oficiālā nosūtīšanas moduļa vienīgā atļautā mērķa adrese būs `MUNICIPALITY_EMAIL`; nodaļas vai darbinieka automātiska izvēle nav paredzēta.

## Elektroniskā parakstīšana

`SIGNING_PROVIDER=mock` aktivizē tikai izstrādes simulāciju. Tā prasa nepārprotamu lietotāja darbību atsevišķā ekrānā, bet nerada kriptogrāfisku parakstu un produkcijas vidē ir aizliegta. `SIGNING_PROVIDER=eparaksts_portal` izvēlas produkcijas adaptera robežu; reālie tīkla izsaukumi jāaktivizē tikai pēc līguma ar LVRTC, HTTPS atgriešanās adreses un servera noslēpumu konfigurēšanas.

Arhitektūra balstīta uz LVRTC dokumentēto sesijas/novirzīšanas plūsmu: [PortalSign integrācijas vadlīnijas](https://developers.eparaksts.lv/docs/portalsign-integration-guidelines) un [SignAPI saskarnes](https://developers.eparaksts.lv/docs/signapi-interfaces). Nekāda parole, API atslēga vai sertifikāts netiek glabāts kodā vai pārlūkā.

## Nosūtīšana un atbildes

Pēc derīga parakstīšanas rezultāta nosūtīšanas serviss izmanto tikai `MUNICIPALITY_EMAIL`. Klienta pieprasījums nevar aizstāt šo saņēmēju vai izvēlēties pašvaldības nodaļu. Vienam parakstītam dokumentam tiek glabāts viens piegādes ieraksts; veiksmīgu nosūtīšanu nevar atkārtot, bet neveiksmīgu — ne vairāk kā `MAX_DELIVERY_ATTEMPTS` reizes un tikai ar administratora darbību.

Ienākošo e-pastu savienotājs nodod webhookam e-pasta `messageId`, `inReplyTo`/`references`, tematu, tekstu un pielikumus. Sistēma sasaista atbildi tikai pēc iepriekš nosūtīta e-pasta thread identifikatora vai atpazīstama `LIM-YYYY-NNNNNN` lietas numura. Neskaidri gadījumi tiek noraidīti, nevis piesaistīti ar AI minējumu. Atbildes termiņš tiek noteikts ar `DEFAULT_RESPONSE_DUE_DAYS` vai administratora darbību; kodā juridisks termiņš nav fiksēts.

## Kabinets un administratora panelis

Pēc pirmā iesnieguma lietotājs saņem drošu saiti uz `/cabinet.html`; marķieris tiek glabāts tikai kā hash datubāzē. Kabinetā redzamas visas ar šo kontu saistītās lietas, dokumentu versijas, avoti, atbildes un auditācijas laika līnija. Administratora panelis ir `/admin.html`; tas prasa `ADMIN_API_TOKEN` un piedāvā tikai pārskatu, filtrus, termiņu un retry darbības — ne parakstīšanu vai apstiprināta dokumenta labošanu.
