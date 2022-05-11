%title: HAHAHA, en Kubernetes operator i Rust
%author: Kent Daleng | http://sad.energy
%date: 2022-05-31

-> # HAHAHA <-

En historie om 
- Kubernetes,
- *Naisjobs*,
- et namespace som fyltes opp av `Pods`
- og et hobbyprosjekt som nå har ansvar for ca. 7000 `Pods` per måned i produksjon

------

-> # Litt om meg <-

- Jeg har jobba i Nav siden 2019.
- Jeg har jobba i *nais* siden starten av 2021, men jobba i team *Flex* før det.

- Relevant for presentasjonen: Jeg er glad i Rust!
- Hadde én enkel Rust-app i produksjon mens jeg jobba i team *Flex*, men den har blitt pensjonert.
  - Ikke fordi den var dårlig, tror jeg, arkitekturendringer gjorde den overflødig!
- Har nå skrevet en annen Rust-app som har endt opp med en del ansvar.
  - Den rydder opp etter "fullførte" *Naisjobs*! (Hva "fullført" betyr spørs hvem du spør)

- Er også diktator av *pdfgen*, men det er en annen historie.

------

-> # Litt om Rust <-

Dette vil ikke være fokuset i presentasjonen, men jeg tenker det er greit å nevne uansett.

- Et generelt programmeringsspråk.
  - Brukes til embedded, native executables, web assembly og sikkert mye annet rart.
- Fokus på ytelse og sikkerhet.
  - Minnesikkerhet uten garbage collector.
  - Uten å gå noe i detalj er en hissig kompilator kanskje den største bidragsyteren til minnesikkerhet. :)
- Har vært “most loved programming language” i StackOverflow’s utviklerundersøkelse siden 2016.

------

-> # Litt om Naisjobs <-

Jeg ønsker å sette en *Naisjob* i kontekst også.

```
apiVersion: nais.io/v1
kind: Naisjob
metadata:
  labels:
    team: aura
  name: sleepy
  namespace: aura
spec:
  image: navikt/perl
  schedule: "0/5 * * * *"
  command: ["perl", "-le", "sleep(15);print 'job complete'"]
```

- En *Naisjob* abstraherer Kubernetes-ressursene `Job` og `CronJob`.
- Dersom `spec.schedule` er definert, vil *Naisjob*-ressursen lage en `CronJob`-ressurs.
- Dersom feltet er udefinert, vil det lages en `Job`-ressurs i stedet.

- En `CronJob`-ressurs vil lage en `Job`-ressurs ved jevne mellomrom (som definert i schedule)
- En `Job`-ressurs vil lage en `Pod`-ressurs.
- En `Pod`-ressurs vil bestå av en eller flere Containere. (*Foreshadowing*)

... Men vi kan sikkert bare ignorere alt mellom *Naisjob* og `Pod` egentlig.

------

-> # Spilleregler på nais <-

> **image**
> Your `/(application|Naisjob)/`'s Docker image location and tag.
> **Type**: string
> **Required**: `true`
? *https://doc.nais.io/naisjob/reference/#image*
? *https://doc.nais.io/nais-application/application/#image*

- I *nais* er det påkrevd at man har én hovedcontainer definert i `spec.image`.
- I motsetning til en *Application*, er en *Naisjob* ment å gjøre seg ferdig.
- Dersom *Naisjob*-ressursen befinner seg i GCP, vil `Pod`-ressursen få en ekstra container: `linkerd-proxy`
- `spec.secureLogs.enabled`, `spec.gcp.sqlInstances` og `spec.vault.sidecar` gjør også noe med antall Containere i `Pod`-ressursene.

*ps*: Man kaller slike ekstra Containere ved siden av hovedcontaineren *sidecars*.

------

-> # Problemet med Kubernetes og sidecars i Naisjob-Pods <-

Kubernetes har ikke noen mekanismer for å fortelle eventuelle *sidecars* når hovedcontaineren er ferdig.
? *https://github.com/kubernetes/kubernetes/issues/25908*

En `Pod` når kun fullført-state når alle Containere har fullført uten feil.
`Pod`-ressurser som ikke fullfører kan leve i clusteret ca. for alltid.

# Kubernetes Operator

En mulig måte å løse dette på er en Kubernetes Operator som rydder opp.

En Kubernetes Operator i *nais* er ofte bare en vanlig *nais* Application.
Applikasjonen får kanskje litt flere rettigheter for å kunne kalle på API-et til Kubernetes, alt etter behov.

------

-> # Ginuudan <-

I september 2021 rulla vi ut vårt første forsøk på en slik Operator.
Vi skrev den i Python, litt fordi vi hadde lyst å se hvordan det ville funke.
Denne typen tilnærming er kanskje et tema for meg som utvikler...

I Python er det mest populære rammeverket for Kubernetes Operators *Kopf*, så vi brukte det.
? *https://github.com/nolar/kopf*

De første ukene virket det som om at alt funka greit, men ting ble verre og verre.

- Rammeverket antar en del av clusteret.
  - Den forsøker å lagre og lese state i clusteret.
  - Om noe ikke stemmer, tør den ikke å gjøre så mye.
- I mange tilfeller ville den ikke gjøre noe som helst.
- Uansett hvor "dum" vi prøvde å gjøre Operatoren gjorde den aldri en perfekt jobb, og mange `Pods` ble liggende i en halvveis ferdig state.

? *https://github.com/nais/ginuudan*

------

-> # Namespacet som ikke ville ta imot opp flere Pods <-

Det at Ginuudan ikke gjorde en god jobb var kjent ganske lenge.
Det ble foreslått å gjøre en større endring, men det ble ikke prioritert så veldig høyt; det var ikke så himla mange som brukte *Naisjobs* uansett.

Men det ble andre boller ei helg tidlig i februar 2022; et annet team begynte å benytte seg av *Naisjobs*.
Teamet satte opp en *Naisjob* med schedule til å kjøre hvert femte minutt.

I et gitt namespace har man lov til å ha 200 `Pods`.
Dersom Operatoren vår ikke rydder opp tar det litt under 17 timer å fylle opp et tomt namespace.
? *https://github.com/nais/teams/blob/master/team-namespaces/templates/teams.yaml#L395-L401*

------

-> # HAHAHA <-

I desember 2021 begynte jeg på et nytt hobbyprosjekt på fagtorsdag; en ny versjon av Ginuudan i Rust.
Fordi jeg hadde lyst å se hvordan det ville funke. :)

Jeg identifiserte ganske raskt *kube-rs* som det beste alternativet for å bygge en enkel Operator.
? *https://github.com/kube-rs/kube-rs*

Sammenlignet med Python-biblioteket *kopf* er vi litt lengre nedi grauten.
Til gjengjeld får man en følelse av at man har bedre kontroll på implementasjonen.

En artig greie med *kube-rs* er at man kan ta nytte av "generics" i Kubernetes API-ene.

```
let client = Client::try_default().await?;
let pods: Api<Pod> = Api::all(client.clone()); // Api<Pod> definerer at vi ønsker Pod-apiet
let jobs: Api<Job> = Api::all(client); // Api<Job> definerer at vi ønsker Job-apiet
```

Deretter kan man bruke funksjonene definert i *Api*
? *https://docs.rs/kube/latest/kube/struct.Api.html*

```
let target_pod: Pod = pods.get("my-pod").await?;
let target_job: Job = jobs.get("my-job").await?;
```

? *https://github.com/nais/hahaha*

------
-> # Men hvordan ser arkitekturen ut? <-

I enkle trekk er det ikke mye mer enn følgende:

- Overvåk `Pod`-ressurser
  - Kubernetes-APIet tillater å abbonnere ved et såkalt `watch`
  - *HAHAHA* benytter seg også av en label-selector.
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

-> # Trial by fire + a series of fortunate events <-

- 9. februar: Jeg presenterte *HAHAHA* for interessenter i *nais*.
  - På dette tidspunktet var applikasjonen ganske funksjonell.
  - Hadde lyst å vise frem hvordan en Operator ser ut i Rust!
  - Applikasjonen ble ikke vurdert som en erstatning for Ginuudan enda.
- 14. februar: Jeg ble gjort oppmerksom på det fulle namespacet.
  - Hobbyprosjektet ble dermed deploya med et feature-flagg samme dag
  - Teamet fikk ting rydda opp ved å gi *HAHAHA* en sjanse!
- 16. februar fikk *HAHAHA* ansvar for alle nye *Naisjobs*.
- De neste dagene ble *HAHAHA* mer raffinert.
- Siden 25. februar har det vært null feil.

------

-> # Hva er forbedringene i forhold til Ginuudan? <- 

Først og fremst, tester! Ginuudan hadde ca. ingen, *HAHAHA* har noen fine integrasjonstester som tester businesslogikken, som kjører i CI. :)

På obervasjonsfronten er *HAHAHA* mye bedre óg: Tallet på avskrudde Containere blir rapportert i Prometheus. 
Dersom ingen Containere har blitt skrudd av innen 15 minutter vil det trigge en alarm på Slack.

*HAHAHA* sender også Events på Kubernetes, slik at hver *Naisjob* får litt info om hva som har blitt gjort.

```
$ kubectl describe pod my-naisjob
# stuff here ...
  Normal  Killing    2m24s  hahaha                                 Shut down container secure-logs-fluentd
  Normal  Killing    2m24s  hahaha                                 Shut down container linkerd-proxy
  Normal  Killing    2m24s  hahaha                                 Shut down container secure-logs-configmap-reload
```

Det ble nevnt 7000 `Pods` i produksjon i første slide.
Om man inkluderer kontrolljobbene mine skrus det av ca. 64000 `Pods` hver måned.

-------

-> # Hva har jeg lært? <- 

Når det gjelder microservices som dette er en refactor fort like kostbar som en hel rewrite.

Det å utnytte fagtorsdager er viktig!
Selv om man kanskje ikke har en kjempekul idé kan man sette av litt tid til å tenke, se litt på andres prosjekter, og kanskje få litt inspirasjon.
Uansett er det gøy å utforske og bygge litt merkompetanse ;)

Å løse et problem med favorittspråket mitt er gøy på kort sikt, men kjedelig på lang sikt.
Det er månedsvis siden jeg sist har jobba med Rust, fordi ting funker dritbra. :(
Jeg kommer til å se om jeg ikke kan pushe litt mer Rust der det passer seg.

------

-> # Takk for meg :) <-

Spørsmål? @ meg på Slack. :)

------