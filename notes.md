%title: HAHAHA, en Kubernetes operator i Rust
%author: Kent Daleng
%date: 2022-05-31

------

-> # Naisjob <-

Det var en gang en naisjob!

- En *Naisjob* er en ressurs som lager `Pods` som kjører enten periodevis eller én gang.
- En `Pod`-ressurs vil bestå av en eller flere Containere.

-------

-> # Litt om meg <-

- Jeg har jobba i Nav siden 2019, men var også innom som sommerstudent i 2018.
- Jeg har jobba i *nais* siden starten av 2021, men jobba i team *Flex* før det.
- Ting jeg er stolt av å ha vært med å lage: 
  - *pdfgen*, 
  - *Wonderwall*,
  - og en greie som rydder opp etter *Naisjobs* i alle clustere.

------

-> # Spilleregler på nais <-

- I *nais* er det påkrevd at man har én hovedcontainer definert i `spec.image`.
- Dersom *Naisjob*-ressursen befinner seg i GCP, vil `Pod`-ressursen få en ekstra container: `linkerd-proxy`
- `spec.secureLogs.enabled`, `spec.gcp.sqlInstances` og `spec.vault.sidecar` gjør også noe med antall Containere i `Pod`-ressursene.

*ps*: Man kaller slike ekstra Containere ved siden av hovedcontaineren *sidecars*.

------

-> # Problemet med Kubernetes og sidecars i Naisjob-Pods <-

Kubernetes har ikke noen mekanismer for å fortelle eventuelle *sidecars* når hovedcontaineren er ferdig.
? *https://github.com/kubernetes/kubernetes/issues/25908*

En `Pod` når kun fullført-state når alle Containere har fullført uten feil.
`Pod`-ressurser som ikke fullfører kan leve i clusteret ca. for alltid.

------

-> # Kubernetes Operator <-

Én mulig måte å løse dette på er en Kubernetes Operator som rydder opp.

En Kubernetes Operator i *nais* er ofte bare en vanlig *nais* Application.
Applikasjonen får rettigheter for å kunne kalle på API-et til Kubernetes, alt etter behov.
Med det har programmet vårt mulighet til å kalle f.eks. `kubectl get pods`, `kubectl exec`.

Andre eksempler på Operators i *nais* er f.eks. Azurerator som oppretter en Application i Azure Active Directory når en Application har `spec.azure.application.enabled` satt.

------

-> # Første forsøk <-

I september 2021 rulla vi ut vårt første forsøk på en slik Kubernetes Operator; Ginuudan.
Vi skrev den i Python, litt fordi vi hadde lyst å se hvordan det ville funke.

I Python er det mest populære rammeverket for Kubernetes Operators *Kopf*, så vi brukte det.

Opplegget slo seg vrangt ganske fort.

- Rammeverket antar en del av clusteret.
  - Den forsøker å lagre og lese state i clusteret.
- I mange tilfeller ville den ikke gjøre noe som helst.
- Uansett hvor "dum" vi prøvde å gjøre Operatoren gjorde den aldri en perfekt jobb, og mange `Pods` ble liggende i en halvveis ferdig state.
- Vi hadde ikke noen tester på overordnet logikk, så alt av "testing" skjedde i clusterne.

------

-> # Namespacet som ikke ville ta imot opp flere Pods <-

Tidlig i februar i år begynte et team å benytte seg av *Naisjobs*.
De satte opp en *Naisjob* med schedule til å kjøre hvert femte minutt.

I et gitt namespace har man lov til å ha 200 `Pods`.
? *https://github.com/nais/teams/blob/master/team-namespaces/templates/teams.yaml#L395-L401*

------

-> # Fagtorsdag <-

Lenge har jeg benytta meg av fagtorsdager til å prøve ting med Rust.
Jeg utforsker hva det kan brukes til, og finner ut om jeg liker det.
(Man kan gjerne si at jeg utøver trivseldrevet utvikling, for jeg er mest produktiv når jeg koser meg.)

For å se om språk og rammeverk egnes har jeg laga noen ting:
- Jeg har laga et par backend-apper (6/10 på trivselskalaen, asynk databasegreier er litt skummelt/vanskelig, men jeg har lyst å prøve igjen)
- Jeg har implementert Tetris 3 eller 4 ganger (8/10, visse spillrammeverk føles ganske gode!)
- En Kubernetes Operator (9/10 kanskje)

------

-> # Rust <-

Det er én ting jeg liker ved Rust: Den hissige kompilatoren.
Jeg anser kompilatorfeil i Rust som noe som kan supplere tester rundt businesslogikk.

I tillegg er kompilatorfeil veldig hjelpsomme. 
Man får vite hvilken variabel som er problemet, hva som er problemet, og hvor problemet oppstår.
I dette tilfellet tar Api eierskap av `client`-variabelen, og på linje 66 er dermed `client` ikke å oppdrive.
Én løsning er å lage to forskjellige klienter, eller klone den ene vi har slik at 39 og 66 får klienten i hver sin instans i minne.

------

-> # HAHAHA <-

I desember 2021 begynte jeg på et nytt hobbyprosjekt på fagtorsdag; en ny versjon av Ginuudan i Rust.

Jeg identifiserte ganske raskt *kube-rs* som det beste alternativet for å bygge en sånn enkel Operator.

Ved hjelp av Rust sin kompilator kunne jeg skrive en ganske "korrekt" applikasjon.
Selv om businesslogikken kanskje ikke var helt på plass i første iterasjon, feila den i alle fall ikke på grunn av null pointers og sånne ting.

-------

-> # Men hva gjør implementasjonen? <-

- Overvåk `Pod`-ressurser
  - `kubectl get pods -w --all-namespaces`
- Finn hovedcontaineren ved å matche Containernavnet med `labels.app`
  - Dette labelet settes av Naiserator, uten den ville det vært ganske vanskelig :)
- Når hovedcontaineren til en `Pod` er ferdig, gå gjennom hver resterende Container og eliminér den
  - Siden det er et fastsatt sett av mulige *sidecars* kan man enkelt hardkode dette
  - Hver *sidecar* har sin egen metode for å skru av

```
pub fn generate() -> BTreeMap<String, Action> {
    let mut actions = BTreeMap::new();
    actions.exec("cloudsql-proxy", "kill -s INT 1");
    actions.exec("vks-sidecar", "/bin/kill -s INT 1");
    actions.exec("secure-logs-configmap-reload", "/bin/killall configmap-reload");
    actions.portforward("linkerd-proxy", "POST", "/shutdown", 4191);
    actions.portforward("secure-logs-fluentd", "GET", "/api/processes.killWorkers", 24444);
    actions
}
```

------

-> # Trial by fire <-

- 9. februar presenterte jeg *HAHAHA* for interessenter i *nais*.
  - På dette tidspunktet var applikasjonen ganske funksjonell.
  - Hadde lyst å vise frem hvordan en Operator ser ut i Rust!
  - Applikasjonen ble ikke vurdert som en erstatning for Ginuudan enda.
- 14. februar ble jeg gjort oppmerksom på det fulle namespacet.
  - Hobbyprosjektet ble dermed gjenoppliva, og deploya med et feature-flagg samme dag
  - Teamet fikk ting rydda opp ved å gi *HAHAHA* en sjanse!
- 16. februar fikk *HAHAHA* ansvar for alle nye *Naisjobs*.
- De neste dagene ble *HAHAHA* mer raffinert.
- Siden 25. februar har vi hatt én alert, som jeg mener kom av større clusterproblemer og ikke angår "korrektheten" til programmet mitt.

------

-> # Hva er forbedringene i forhold til Ginuudan? <- 

Først og fremst, tester! Ginuudan hadde ca. ingen, *HAHAHA* har noen fine integrasjonstester som tester businesslogikken, som også kjører i CI. :)

På obervasjonsfronten er *HAHAHA* mye bedre óg: Tallet på avskrudde Containere blir rapportert i Prometheus.
Dersom ingen Containere har blitt skrudd av innen 15 minutter vil det trigge en alert på Slack.

*HAHAHA* sender også Events på Kubernetes, slik at hver *Naisjob* får litt info om hva som har blitt gjort.

```
$ kubectl describe pod my-naisjob
# stuff here ...
  Normal  Killing    2m24s  hahaha                                 Shut down container secure-logs-fluentd
  Normal  Killing    2m24s  hahaha                                 Shut down container linkerd-proxy
  Normal  Killing    2m24s  hahaha                                 Shut down container secure-logs-configmap-reload
```

Om man inkluderer kontrolljobbene mine skrus det av ca. 64000 `Pods` hver måned, men ca. 7000 reelle jobber.

-------

-> # Hva har jeg lært? <- 

Når det gjelder microservices som dette er en refactor fort like kostbar som en hel rewrite.

Det å utnytte fagtorsdager er viktig!
Selv om man kanskje ikke har en kjempekul idé kan man sette av litt tid til å tenke, se litt på andres prosjekter, og kanskje få litt inspirasjon.
Uansett er det gøy å utforske og bygge litt "merkompetanse" ;)

Å løse et problem med favorittspråket mitt er gøy på kort sikt, men kjedelig på lang sikt.
Det er månedsvis siden jeg sist har jobba med Rust, fordi ting funker dritbra. :(

------

-> # Takk for meg :) <-

Spørsmål? @ meg på Slack. :)

------