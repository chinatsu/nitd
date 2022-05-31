%title: HAHAHA, en Kubernetes operator i Rust
%author: Kent Daleng / https://github.com/chinatsu/nitd
%date: 2022-05-31

------

-> # Naisjob <-

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

------

-> # Litt om meg <-

- Jeg har jobba i Nav siden 2019, men var også innom som sommerstudent i 2018.
- Jeg har jobba i *nais* siden starten av 2021, men jobba i team *Flex* før det.
- Ting jeg er stolt av å ha vært med å lage: 
  - *pdfgen*, 
  - *Wonderwall*,
  - og en greie som rydder opp etter *Naisjobs* i alle clustere.


------

-> # Spilleregler på nais <-

> **image**
> Your `Naisjob`'s Docker image location and tag.
> **Type**: `string`
> **Required**: `true`
? *https://doc.nais.io/naisjob/reference/#image*

- Dersom *Naisjob*-ressursen befinner seg i GCP, vil `Pod`-ressursen få en ekstra container: `linkerd-proxy`
- `spec.secureLogs.enabled`,
- `spec.gcp.sqlInstances` og 
- `spec.vault.sidecar` gjør også noe med antall Containere i `Pod`-ressursene.

------

-> # Problemet med Kubernetes og sidecars i Naisjob-Pods <-

Kubernetes har ikke noen mekanismer for å fortelle eventuelle *sidecars* når hovedcontaineren er ferdig.
? *https://github.com/kubernetes/kubernetes/issues/25908*


------

-> # Kubernetes Operator <-

Én mulig måte å løse dette på er en Kubernetes Operator som rydder opp.

Andre eksempler på Operators som finnes i *nais* er f.eks. 
Azurerator som oppretter en Application i Azure Active Directory når en Application har `spec.azure.application.enabled` satt.

------

-> # Første forsøk <-

I september 2021 rulla vi ut vårt første forsøk en slik Kubernetes Operator; Ginuudan.

Det mest populære rammeverket for Kubernetes Operators i Python er *Kopf*.
? *https://github.com/nolar/kopf*

? *https://github.com/nais/ginuudan*

------

-> # Namespacet som ikke ville ta imot opp flere Pods <-

Tidlig i februar i år begynte et team å benytte seg av *Naisjobs*.
Teamet satte opp en *Naisjob* med schedule til å kjøre hvert femte minutt.

I et gitt namespace har man lov til å ha 200 `Pods`
Dersom Operatoren vår ikke rydder opp tar det litt under 17 timer å fylle opp et tomt namespace.
? *https://github.com/nais/teams/blob/master/team-namespaces/templates/teams.yaml#L395-L401*

------

-> # Fagtorsdag <-

Lenge har jeg benytta meg av fagtorsdager til å prøve ting med Rust.

For å se om språk og rammeverk egnes har jeg laga noen ting:
- Et par backend-apper (6/10)
- Tetris (8/10!)

------

-> # Rust <-

```
error[E0382]: use of moved value: `client`
  --> src/main.rs:66:48
   |
37 |     let client = Client::try_default().await?;
   |         ------ move occurs because `client` has type `kube::Client`, which does not implement the `Copy` trait
38 | 
39 |     let pods: Api<Pod> = Api::all(client);
   |                                   ------ value moved here
...
66 |             Context::new(reconciler::Data::new(client, reporter, actions)),
   |                                                ^^^^^^ value used here after move
    
For more information about this error, try `rustc --explain E0382`.
error: could not compile `hahaha` due to previous error
```

------

-> # HAHAHA <-

I desember 2021 begynte jeg på et nytt hobbyprosjekt på fagtorsdag; en ny versjon av Ginuudan i Rust.

Jeg identifiserte ganske raskt *kube-rs* som det beste alternativet for å bygge en sånn enkel Operator.
? *https://github.com/kube-rs/kube-rs*

```
let client = Client::try_default().await?;
let pods: Api<Pod> = Api::all(client.clone()); // Api<Pod> definerer at vi ønsker Pod-apiet
let jobs: Api<Job> = Api::all(client); // Api<Job> definerer at vi ønsker Job-apiet
 
let target_pod: Pod = pods.get("my-pod").await?;
let target_job: Job = jobs.get("my-job").await?;
```

? *https://github.com/nais/hahaha*

-------

-> # Men hva gjør implementasjonen? <-

- Overvåk `Pod`-ressurser
- Finn hovedcontaineren ved å matche Containernavnet med `labels.app`
- Når hovedcontaineren til en `Pod` er ferdig, gå gjennom hver resterende Container og eliminér den

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
- 14. februar ble jeg gjort oppmerksom på det fulle namespacet.
- 16. februar fikk *HAHAHA* ansvar for alle nye *Naisjobs*.
- De neste dagene ble *HAHAHA* mer raffinert.
- Siden 25. februar har vi hatt én alert.

------

-> # Hva er forbedringene i forhold til Ginuudan? <- 

Først og fremst, tester! Ginuudan hadde ca. ingen,
*HAHAHA* har noen fine integrasjonstester som tester businesslogikken, som også kjører i CI. :)

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

-------

-> # Hva har jeg lært? <- 

Når det gjelder microservices som dette er en refactor fort like kostbar som en hel rewrite.

Det å utnytte fagtorsdager er viktig!

Å løse et problem med favorittspråket mitt er gøy på kort sikt, men kjedelig på lang sikt.
Det er månedsvis siden jeg sist har jobba med Rust, fordi ting funker dritbra. :(

------

-> # Takk for meg :) <-

Spørsmål? @ meg på Slack. :)

------