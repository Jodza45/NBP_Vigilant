# Arhitektura Vigilant Backend-a (Clean Architecture)

Vigilant backend je dizajniran primenom **Clean Architecture** (Čiste arhitekture) obrazaca, sa jasnom separacijom odgovornosti između slojeva. Projekat je podeljen na četiri glavna projekta (sloja), koji komuniciraju u jednom smeru ka unutra: **API -> Application -> Domain**, i **Infrastructure -> Application**. Komunikacija unutar slojeva i između njih često koristi **CQRS** (Command Query Responsibility Segregation) šablon implementiran preko **MediatR** biblioteke. Backend kao glavno skladište podataka koristi **Neo4j**, graf bazu podataka, kako bi efikasno mapirao povezanost korisnika, računa i transakcija u cilju detekcije AML (Anti-Money Laundering) obrazaca (pranja novca).

U nastavku je detaljan pregled svakog sloja, njegovih foldera i funkcionalnosti.

---

## 1. Vigilant.Domain (Domen)

Ovaj sloj predstavlja srce aplikacije i **ne zavisi ni od jednog drugog sloja** niti tehnologije. Tu se definišu osnovni entiteti (strukture podataka) i enumeracije, odnosno sama pravila i rečnik poslovnog domena.

### Šta se u njemu nalazi:
* **`Alerts/` (folder)**
  * **Namena:** Modeli i definicije vezane za alarme o potencijalnom pranju novca.
  * **Sadržaj:**
    * `AlertNode.cs`: Klasa koja predstavlja čvor alarma unutar grafa.
    * `AlertStatus.cs`: Enumeracija koja definiše stanja alarma (npr. `Open`, `InProgress`, `Closed`).
    * `AlertAuditEntry.cs`: Model zapisa za praćenje promena (istorijata) na određenom alarmu (koji analitičar ga je promenio, kada i zašto).
    * `AmlAlertTypes.cs`: Konstante koje definišu tipove pranja novca (npr. *Smurfing*, *CircularFlow*, *VelocityChain*).
* **`Graph/` (folder)**
  * **Namena:** Entiteti koji direktno predstavljaju čvorove u graf bazi.
  * **Sadržaj:** `AccountNode.cs`, `ClientNode.cs`, `DeviceNode.cs`, `IpAddressNode.cs`, `TransactionNode.cs`. Ove klase mapiraju objekte iz stvarnog sveta (računi, klijenti, transakcije, IP adrese, uređaji) koji se prate u cilju zaštite.

### Funkcionalnost:
Služi kao ugovor i definicija tipova za sve ostale slojeve. Ne radi nikakvu eksternu komunikaciju.

---

## 2. Vigilant.Application (Aplikacija)

Ovo je sloj u kom se nalazi srž **poslovne logike** ("use-cases"). Ovaj sloj definiše *šta* sistem radi. Aplikacioni sloj definiše interfejse (kao što su repozitorijumi) koje onda Infrastrukturni sloj implementira.

### Šta se u njemu nalazi:
* **`Alerts/` (folder)**
  * **Namena:** Rukovanje alarmima koristeći CQRS (Upiti i Komande).
  * **Sadržaj:** Sadrži podfoldere za konkretne use-case radnje kao što su `GetAlertById`, `GetAlerts`, `UpdateAlertStatus`. U ovim podfolderima su klase za upite i komande, kao i njihovi handler-i.
  * U podfolderu **`Detection/`** nalazi se `IAmlDetectionService` interfejs i njegova implementacija `AmlDetectionService`.
  * **Funkcionalnost:** `AmlDetectionService` poziva pravila iz graf baze nad novom transakcijom. Ukoliko baci alarm (npr. uočen je Circular Flow), on poziva `IAlertPublisher` da objavi taj alarm sistemu.
* **`Clients/` (folder)**
  * **Namena:** Poslovna logika vezana za klijente.
  * **Sadržaj:** Podfolder **`Risk/`** sadrži `RiskScoreService` i modele. Služi za ažuriranje i proračun faktora rizika klijenta (*Risk Score*) na osnovu detektovanih sumnjivih radnji (npr. povećaj rizik ako je uhvaćen u Smurfing-u).
* **`Common/` (folder)**
  * **Namena:** Deljeni interfejsi i DTO (Data Transfer Objects) modeli za prenos podataka.
  * **Interfejsi koje definiše:**
    * `IAlertPublisher`: Interfejs za broadcast (slanje) alarma ka korisnicima.
    * **`INeo4jRepository`**: Ključni ugovor za rad sa graf bazom. Definiše metode za upis transakcija, dobijanje entiteta i izvršavanje specifičnih AML detekcija (npr. `FindCircularFlowsAsync`, `FindSmurfingAsync`).
* **`Graph/` (folder)**
  * **Namena:** Upiti (`GetGraphQuery`, `GetGraphOverviewQuery`) koji se koriste za povlačenje podataka potrebnih frontend-u kako bi nacrtao graf (mrežu transakcija i korisnika).
* **`Transactions/` (folder)**
  * **Namena:** Orkestracija dolaznih transakcija.
  * **Sadržaj:** Podfolder **`ProcessTransaction/`** sadrži `TransactionProcessorCommand` i njegov `Handler`.
  * **Funkcionalnost:** Ovo je najvažnija putanja. Kada stigne transakcija, Handler (koristeći `INeo4jRepository`) upisuje transakciju u bazu. Odmah nakon upisa poziva se `IAmlDetectionService` da izvrši proveru. Zatim se na osnovu pronađenih alarma ažuriraju nivoi rizika vlasnika računa. Ovdje se nalazi i folder **`SeedTransactions/`** sa servisima koji se koriste za veštačko upisivanje demo podataka za potrebe prezentacije.

---

## 3. Vigilant.Infrastructure (Infrastruktura)

Zadužen je za eksternu komunikaciju – konkretno bazu podataka. On implementira interfejse definisane u `Vigilant.Application` sloju.

### Šta se u njemu nalazi:
* **`Graph/` (folder)**
  * **Sadržaj:** Klasa `Neo4jRepository`.
  * **Funkcionalnost:** Implementira interfejs `INeo4jRepository`. Koristi ugrađeni drajver graf baze (`IDriver`) za komunikaciju sa instancom Neo4j. Ovde se nalaze masivni, kompleksni i sirovi **Cypher** upiti (jezik za upit nad graf bazama). Repozitorijum radi klasične stvari (čitanje/upis alarma i čvorova) ali i izuzetno teške pretrage (npr. upiti za pronalazak putanja novca koje ukazuju na "Kružne tokove" (*Circular flows*), "Razbacivanje sredstava" (*Rapid Fan Out*) ili "Cepače" (*Smurfing*)).
* **`Options/` (folder)**
  * **Sadržaj:** Klasa `Neo4jOptions`.
  * **Funkcionalnost:** Implementacija Options šablona. Mapira postavke vezane za konekciju sa bazom (URL, username, password) iz `appsettings.json` fajla.
* **`Seed/` (folder)**
  * **Sadržaj:** Klasa `DataSeederService`.
  * **Funkcionalnost:** Sadrži logiku koja komunicira sa bazom da bi izgenerisala velike setove probnih podataka (račune klijenata i hiljade povezanih transakcija) kako bi baza bila obogaćena testnim podacima.
* **`DependencyInjection.cs`**
  * Sadrži kod koji u .NET DI Container (Kontejner za zavisnosti) registruje ove servise (npr. `services.AddScoped<INeo4jRepository, Neo4jRepository>()`).

---

## 4. Vigilant.Api (API)

Ovo je **Presentation Layer** (Prezentacioni sloj). On eksponira funkcionalnosti backend-a spoljnim sistemima i frontendu pomoću REST API arhitekture i WebSockets-a. Apsolutno nema nikakvu kompleksnu poslovnu logiku, već isključivo prosleđuje podatke.

### Šta se u njemu nalazi:
* **`Controllers/` (folder)**
  * **Namena:** Definisanje REST HTTP endpoint-ova (rute do kojih se može doći HTTP GET, POST itd. zahtevima).
  * **Sadržaj:**
    * `AlertsController.cs`: Za čitanje AML alarma i promenu njihovog statusa (koristeći MediatR instancu `ISender` za prosleđivanje poziva u Application sloj).
    * `TransactionsController.cs`: Sadrži rutu `/process` koja prima POST zahteve novih transakcija iz bankarskog sistema i `/seed` za okidanje generisanja demo podataka.
    * `ClientsController.cs` i `GraphController.cs`: Za dohvatanje specifičnih podataka klijenata i vizuelne strukture grafa za mapu transakcija na frontendu.
* **`Contracts/` (folder)**
  * **Namena:** API modeli i zahtevi. Ovdje se nalaze Request i Response strukture podataka (npr. `ProcessTransactionRequest`, `UpdateAlertStatusRequest`). Služe isključivo za serijalizaciju i deserijalizaciju JSON formata koji šalje i prima frontend klijent.
* **`Hubs/` (folder)**
  * **Namena:** Rad u realnom vremenu.
  * **Sadržaj:** Sadrži `AlertsHub.cs` klasu koja nasleđuje SignalR `Hub`. Klijenti (React/Angular aplikacije) se povežu na ovaj WebSocket kanal kako bi momentalno dobijali obaveštenja (push notifications) čim analitika uhvati novo pranje novca.
* **`Services/` (folder)**
  * **Namena:** Infrastrukturna implementacija vezana isključivo za presentation sloj.
  * **Sadržaj:** `SignalRAlertPublisher`. Ova klasa **implementira `IAlertPublisher`** iz Application sloja. Zašto se nalazi u API sloju? Zato što šalje poruke direktno kroz klijentske konekcije SignalR hub-a (`IHubContext<AlertsHub>`). Na ovaj način, kada `AmlDetectionService` detektuje problem, on pozove Publisher, a ovaj iz API-ja prosledi podatke u realnom vremenu frontendu, potpuno asinhrono.
* **`Program.cs` i `appsettings.json`**
  * Služi za podizanje same ASP.NET Core aplikacije. Ovde se postavlja Middleware, vrši se spajanje svih zavisnosti, konfigurišu CORS pravila, postavlja Swagger dokumentacija i konfigurišu putanje za kontrolere i SignalR hub. U fajlovima `appsettings` se nalaze svi privatni ključevi i konfiguracije okruženja.

### Kako komunikacija teče (Primer procesiranja transakcije):
1. **API sloj:** Angular šalje JSON podatke o transakciji na `/api/transactions/process`.
2. **API sloj:** `TransactionsController` prihvata podatke kroz `ProcessTransactionRequest` i prosleđuje ih instanci MediatR-a.
3. **Application sloj:** `TransactionProcessorCommandHandler` prihvata zahtev, proverava ispravnost. 
4. **Application <-> Infrastructure sloj:** Handler poziva `INeo4jRepository.WriteTransactionGraphAsync()`. Klasa `Neo4jRepository` u *Infrastructure* sloju stvarno komunicira sa Neo4j bazom da sačuva te podatke.
5. **Application <-> Infrastructure sloj:** Handler poziva `IAmlDetectionService` da ispita pravila, a on poziva masivne pretrage u `Neo4jRepository` (npr. *Smurfing* detekcija).
6. **Application -> API sloj:** Ako je detektovan prekršaj, poziva se `IAlertPublisher`, kojeg implementira `SignalRAlertPublisher` u API sloju koji šalje WebSocket notifikaciju frontendu na ekran!
7. **Application -> API sloj:** Handler vraća finalni rezultat nazad do MediatR-a koji iz kontrolera šalje HTTP 200 OK klijentu.
