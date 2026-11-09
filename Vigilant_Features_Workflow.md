# Tokovi podataka (Workflows) klijentskih funkcionalnosti

U nastavku je detaljan opis svih funkcionalnosti sa kojima korisnik (analitičar) na klijentskoj strani (frontend-u) može da interreaguje. Za svaki klik ili akciju, opisan je tačan tok podataka (*workflow*) i objašnjenje šta, gde i zašto backend radi.

---

## 1. Prikaz glavne mreže / vizuelizacija celog sistema
**Korisnička akcija:** Korisnik otvara početnu stranicu (Dashboard) aplikacije. Ekran prikazuje veliku D3.js mrežu povezanih klijenata, transakcija i IP adresa.
**API Endpoint:** `GET /api/graph/overview?nodeLimit=250`

**Workflow na backendu:**
1. Frontend šalje GET zahtev `GraphController`-u kako bi zatražio grafičke podatke.
2. Kontroler formira i šalje `GetGraphOverviewQuery` preko MediatR-a.
3. Aplikacioni Handler preuzima upit i poziva `Neo4jRepository.GetGraphOverviewAsync()`.
4. **Baza (Neo4j):** Pokreće se specifičan Cypher upit koji pretražuje sve čvorove baze (Klijent, Račun, Transakcija, Uređaj, IP adresa) i njihove međusobne veze. Zbog performansi pregledača, upit ograničava broj vraćenih entiteta (npr. maksimalno 250 čvorova).
5. Podaci iz baze (čvorovi i veze) se transformišu u zajednički `EntityGraphDto` objekat i vraćaju kao JSON.
6. **Zašto:** Ovo je primarni alat analitičara za vizuelni pregled celokupnog stanja u banci.

---

## 2. Dubinska analiza jednog IBAN-a (Account Graph)
**Korisnička akcija:** Analitičar na grafikonu ili u tabeli klikne na specifičan bankovni račun (IBAN) kako bi istražio sa kim je on sve povezan.
**API Endpoint:** `GET /api/graph/accounts/{iban}?depth=4`

**Workflow na backendu:**
1. Frontend gađa `GraphController` prosleđujući rutu sa IBAN-om i opcionim parametrom `depth` (koliko duboko u mrežu želi da uđe - npr. "prikaži mi transakcije onoga kome je ovaj račun poslao novac").
2. Kontroler šalje `GetEntityGraphQuery`.
3. Handler delegira zahtev do `Neo4jRepository.GetEntityGraphAsync()`.
4. **Baza (Neo4j):** Cypher upit traži tačno taj račun po IBAN-u, a onda traži *putanje (paths)* koje idu od njega do daljine od $depth veza (`[0..depth]`). Skuplja sve uređaje, adrese i transakcije kroz koje je taj korisnik vršio uplate.
5. JSON odgovor se vraća, a frontend ponovo iscrtava graf samo za taj uži, specifičan ekosistem oko odabranog računa.

---

## 3. Tabela svih sumnjivih alarma (Alerts Dashboard)
**Korisnička akcija:** Korisnik prelazi na stranicu "Alerts" i gleda tabelu i možda bira status iz dropdown-a (npr. prikaži samo alarme koji su `InProgress`).
**API Endpoint:** `GET /api/alerts?status={status}`

**Workflow na backendu:**
1. Frontend poziva `AlertsController` prosleđujući parametre za filtriranje.
2. Formira se `GetAlertsQuery` koji odlazi u Application sloj.
3. Poziva se `Neo4jRepository.GetAlertsAsync()`.
4. Baza traži sve `Alert` čvorove koji odgovaraju traženom statusu i vremenskom okviru. 
5. Podaci se parsiraju i vraćaju frontendu kao lista objekata (koji uključuju težinu alarma, poruku o tome šta se desilo npr. "Smurfing detected" i tačno vreme kada se to dogodilo).

---

## 4. Rešavanje i menjanje statusa AML Alarma (Update Alert Status)
**Korisnička akcija:** Korisnik otvara detalje nekog alarma, u polje "Komentar" upisuje *"Zatvorili smo korisnički račun, ovo je definitivan pokušaj pranja novca"* i pritisne dugme da promeni status alarma u `Closed`.
**API Endpoint:** `PATCH /api/alerts/{id}/status`

**Workflow na backendu:**
1. Frontend šalje PATCH zahtev sa JSON telom koje sadrži `NewStatus`, `AnalystName` i `Comment` prema `AlertsController`-u.
2. Kontroler inicira `UpdateAlertStatusCommand`.
3. Handler komande prvo koristi bazu (`GetAlertByIdAsync`) da potvrdi da taj alarm uopšte postoji.
4. Ukoliko postoji, sistem **kreira novi `AlertAuditEntry`** (zapis za istorijat) gde upisuje ime analitičara, stari status, novi status i napisan komentar, zajedno sa trenutnim UTC vremenom.
5. Ostatak audit log-a se serijalizuje u JSON niz i baza čuva taj promenjeni status i ažuriran istorijat pomoću komande `Neo4jRepository.UpdateAlertStatusAsync()`.
6. **Zašto:** Po regulativama banaka (Compliance), svaki sumnjivi slučaj se mora revidirati, i ne sme se prosto obrisati greška već uvek mora postojati log *ko* je odlučio da se alarm zanemari ili potvrdi i *zašto*.

---

## 5. Uvid u nivo rizika konkretnog klijenta (Risk Score Profil)
**Korisnička akcija:** Korisnik klikne na informaciju o klijentu da mu pogleda lični profil i trenutnu visinu bezbednosnog rizika.
**API Endpoint:** `GET /api/clients/{id}/risk`

**Workflow na backendu:**
1. Frontend traži rizik od `ClientsController`-a.
2. Kontroler poziva pomoćni `RiskScoreService.GetAsync()` metod iz Application sloja.
3. Ovaj servis vrši `Neo4jRepository.GetClientRiskScoreAsync()`.
4. Upit traži `Client` čvor na osnovu ID-a i iščitava njegov interni properti `RiskScore`. 
5. **Zašto radi:** Svaki klijent kreće sa rizikom 0. Ako uhvatimo klijenta u sumnjivim akcijama (npr. Kružni tokovi uplatnica), sistem sam povećava ovaj broj u bazi kako bi na klijentu uvek stajao taj "prljavi trag". Frontend na osnovu ove brojke (0 do 100) najčešće boji čvor korisnika (zeleno, narandžasto ili crveno).

---

## 6. Prijem Real-time notifikacija (WebSockets / SignalR)
**Korisnička akcija:** Korisnik je upravo otvorio aplikaciju na frontendu i pregleda podatke. Ništa ne radi, ali mu odjednom u uglu iskoči notifikacija (Toast): *"High Severity Alert: Smurfing Detected!"*
**Komunikacioni kanal:** `ws://.../hubs/alerts` (WebSocket konekcija)

**Workflow na backendu:**
1. Kada se frontend učita, automatski se otvara WebSocket veza sa API slojem ka `AlertsHub`-u.
2. Klijent se ubacuje u virtuelnu grupu (tzv. `aml-alerts` grupa).
3. U isto to vreme, ako prođe transakcija koja uzrokuje prekršaj, Application sloj `AmlDetectionService` to oseti i poziva `IAlertPublisher`.
4. Taj interfejs je implementiran kroz `SignalRAlertPublisher` klasu unutar API sloja. On automatski *push*-uje taj događaj svim aktivnim korisnicima koji slušaju preko Websocket-a. Frontend uhvati taj događaj i iscrta popup poruku bez potrebe da se stranica osveži.

---

## 7. Izvršavanje bankarske transakcije (Process Transaction)
**Korisnička akcija:** Kroz backend ili simulator neko pošalje pravu bankarsku transakciju (novac prolazi).
**API Endpoint:** `POST /api/transactions/process`

**Workflow na backendu (Najteži i najkompleksniji na sistemu):**
1. Šalje se JSON sa svim podacima (IBAN sa kog ide novac, IBAN primaoca, Količina, IP adresa klijenta sa kog šalje, Fingerprint brauzera...).
2. `TransactionsController` zaprima podatke i formira instancu `TransactionProcessorCommand` komande (CQRS obrazac) i prosleđuje MediatR-u.
3. `TransactionProcessorCommandHandler` prihvata ovo i obavlja potpunu proveru. Prvo validira ispravnost (IBAN pošiljaoca ne sme biti isti kao IBAN primaoca, iznos > 0).
4. **Upis u graf:** Pokreće se `Neo4jRepository.WriteTransactionGraphAsync`. Ovo kreira novi `Transaction` čvor, dodaje vezu pošiljaoca koji ga je poslao `[:SENT]` i primaoca `[:RECEIVED_BY]`. Spaja i podatke za IP i Uređaj.
5. **Provera Pranja Novca:** Nakon upisa, handler zove `AmlDetectionService.EvaluateAndPublishAsync()`.
   * Servis pokreće Cypher upite baze koji idu retroaktivno kroz graf i gledaju ponašanje onih koji su povezani sa ovom upravo pridošlom transakcijom!
   * Ako uhvati pravila poput `Smurfing` ili `VelocityChain`, on generiše novu instancu alarma.
6. **Slanje alarma analitičaru:** `AmlDetectionService` prosleđuje alarme u `SignalRAlertPublisher` kako bi se pojavio popup na ekranu svih aktivnih analitičara.
7. **Povećavanje rizika klijenta:** Nakon pronalaska alarma, handler komande prepoznaje koje pravilo je prekršeno i dodaje poene rizika korisnicima preko `Neo4jRepository.UpdateClientRiskScoreAsync`. Slanje npr. kružnih transakcija povećaće klijentov Risk Score za +30.
8. Kraj.

---

## 8. Generisanje demo podataka (Seed Data)
**Korisnička akcija:** Korisnik klikne dugme "Seed Database" ili "Generate Demo Data" kako bi aplikacija generisala "prljave" podatke i testirala sistem.
**API Endpoint:** `POST /api/transactions/seed`

**Workflow na backendu:**
1. Frontend šalje zahtev `TransactionsController`-u.
2. Kontroler kontaktira `DataSeederService` (koji stoji u Infrastructure sloju).
3. Servis ubacuje na hiljade sintetičkih i lažnih `Client` i `Account` i `Transaction` entiteta u bazu. Njegov glavni cilj je ne samo da ispuni bazu podacima, već i da **veštački formira AML tipologije** u obliku grafa. 
   - Npr. on programirano kreira mrežu od 5 računa gde prvi račun pošalje svakom od ostala 4 određenu svotu novca, u cilju da namesti (simulira) pranje novca tipa "Rapid Fan Out", kako bi detekcioni sistem backend-a bio uspešno istestiran.
4. Ovi kreirani demo podaci se trajno upisuju u Neo4j graf bazu kako bi bili spremni za testiranje nad Dashboard-om (endpoint br. 1 i 2).
