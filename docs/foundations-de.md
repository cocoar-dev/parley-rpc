# The Parley Protocol — Grundlagen

| | |
|---|---|
| **Status** | Entwurf · Grundlagen |
| **Fassung** | 0.1 |
| **Datum** | 2026-09-28 |
| **Steward** | Cocoar |
| **Slug** | `parley-rpc` |
| **Sprache** | Deutsch (Diskussionsgrundlage — die Spezifikation selbst wird Englisch) |

> **Zusammenfassung.** Parley ist ein Protokoll für bidirektionale Echtzeit-Kommunikation zwischen
> einem Server und seinen Clients: typisierte Aufrufe in beide Richtungen, Item-Streams,
> Byte-Kanäle und ein vollständiger Credential-Lebenszyklus — über einen Transport, der beim
> Verbindungsaufbau *verhandelt* statt vorausgesetzt wird. Dieses Dokument legt die Grundlagen:
> Begriffe, Axiome, das Fähigkeitsmodell, die Invarianten, die Sprachschnittmenge und die offenen
> Entwurfsfragen. Es ist die Diskussionsbasis vor der eigentlichen Spezifikation, kein Wire-Format.

---

## § 1 Zielbild

Ein Parley ist historisch das Gespräch unter weißer Flagge: Zwei Parteien sichern einander freies
Geleit zu und reden dann — Forderungen, Botschaften, Antworten, in beide Richtungen, solange die
Flagge steht. Das Protokoll übernimmt diese Struktur wörtlich.

Eine Verbindung beginnt mit einer **Verhandlung**: Welchen Transport können beide Seiten, welches
Serialisierungsformat, welche Fähigkeiten? Sie erhält **Geleit**: Credentials werden geprüft,
herausgefordert, erneuert — nicht einmal am Anfang, sondern für die gesamte Lebensdauer. Und dann
findet das **Gespräch** statt: Beide Seiten bieten einander Methoden an und rufen sie auf, beide
Seiten senden Streams, beide Seiten übertragen Dateien. Der Server ist dabei kein Antwortautomat —
er ergreift von sich aus das Wort, ruft Methoden seiner Clients auf und erwartet Ergebnisse.

Das Protokoll heißt nach dem Gespräch, nicht nach der Einigung: *to parley* kommt von *parler* —
sprechen. Die Verhandlung ist das, was dieses Gespräch von einem rohen Socket unterscheidet; das
Gespräch selbst ist der Inhalt.

### Warum es dieses Protokoll braucht

Der Raum ist besetzt, aber die Kombination ist frei: Request/Response-Frameworks kennen kein
serverseitig initiiertes, typisiertes RPC zum Browser. Socket-Bibliotheken kennen weder Contracts
noch Credential-Lebenszyklen noch Dateitransfer. Peer-to-Peer-Medienprotokolle lösen ein anderes
Problem. Niemand bedient *typisierte bidirektionale Aufrufe + verhandelte Transports +
Auth-Lebenszyklus + Datei-Transfer + Item-Streaming*, browser-tauglich und über Sprachgrenzen
hinweg, als ein Ganzes. Parley besetzt genau diese Lücke — und nichts darüber hinaus (§ 9).

---

## § 2 Begriffe

| Begriff | Bedeutung |
|---|---|
| **Parley** | Eine verhandelte, beglaubigte, bidirektionale Sitzung zwischen genau zwei Peers. Beginnt mit der Verhandlung, endet mit Abschied oder Abbruch. |
| **Peer** | Eine Seite eines Parley. Die Rollen *Initiator* (Client) und *Acceptor* (Server) unterscheiden sich nur im Verbindungsaufbau und in der Adressierung — nicht in der Fähigkeit, Aufrufe zu beginnen. Der Acceptor ist eine logische Rolle: physisch kann eine Flotte von Knoten dahinterstehen (§ 4). |
| **Contract** | Eine benannte Menge von Methoden, die ein Peer dem anderen anbietet. Wire-Namen sind logisch und sprachneutral; kein Contract-Name enthält jemals einen Bezeichner einer Wirtssprache. |
| **Call / Tell** | Ein *Call* erwartet ein Result oder einen Fault; ein *Tell* ist fire-and-forget. Beide existieren in beide Richtungen. |
| **Stream** | Eine geordnete Folge typisierter Items mit explizitem Ende, in eine Richtung, jederzeit abbrechbar. |
| **Channel** | Ein Byte-Kanal für große Nutzlasten (Dateien). Ob er durch die Frame-Verbindung reist oder über eine Seitenverbindung, ist offen (Q-9) — in jedem Fall gehört er dem Parley: gleiche Identität, gleiche Autorisierung, gleiches Ende. |
| **Reference** | Ein Argument, das kein Wert ist, sondern ein Verweis: ein Abbruch-Token, ein Stream, ein Channel. References sind eigene Wire-Typen und niemals aus der Form eines Wertes zu erraten. |
| **Conduct** | Das Geleit: der Credential-Lebenszyklus eines Parley — Vorlage, Prüfung, Herausforderung, Erneuerung, Ablauf, Entzug. |
| **Muster** | Das Verzeichnis aller laufenden Parleys eines Acceptors — die Musterrolle der Flotte: Identität, Principal, Metadaten, Gruppenzugehörigkeiten je Parley. Grundlage jeder gezielten Adressierung (§ 4); Arbeitsname unter Q-7. |
| **Profile** | Ein benanntes Fähigkeitsbündel, das ein Peer implementiert und in der Verhandlung anmeldet (§ 8). |
| **Frame** | Die kleinste Nachrichteneinheit des Protokolls. Frames sind transportunabhängig; ein Transport schuldet nur ihre geordnete, vollständige Zustellung. |

---

## § 3 Axiome

Sieben Festlegungen, die allem Weiteren vorausgehen. Änderungen hier sind Änderungen am Wesen des
Protokolls. Schlüsselwörter MUST / MUST NOT im Sinne von RFC 2119.

### A-1 — Die Spezifikation ist die Wahrheit

Parley ist ein Dokument plus eine Konformitätssuite. Jede Implementierung ist gleichrangig; keine
ist „die Referenz". Verhält sich eine Implementierung anders als die Spezifikation, ist die
Implementierung falsch — auch die erste, auch die älteste. Streit wird an der Suite entschieden,
nicht am Dienstalter.

### A-2 — Die Schnittmenge regiert die Semantik, nicht die Oberfläche

Eine Fähigkeit darf nur dann Protokoll-Semantik werden, wenn sie in allen Zielsprachen (§ 7)
implementierbar ist. Die SDK-Oberflächen dagegen MUST NOT auf den kleinsten gemeinsamen Nenner
nivelliert werden: eine Wire-Semantik, viele Idiome.

### A-3 — Geschlossene Welt

Das Protokoll setzt zur Laufzeit weder Typ-Introspektion noch Code-Erzeugung voraus. Alles
Aufrufbare ist zur Bauzeit bekannt oder explizit registriert. Wire-Namen sind logisch; kein Frame
trägt jemals den Typ-Bezeichner einer Wirtssprache.

### A-4 — Die Verhandlung endet nie

Transport, Format, Fähigkeiten und Credentials werden beim Aufbau verhandelt und bleiben
verhandelbar: Herausforderung und Erneuerung von Credentials, Neuverhandlung nach
Verbindungsverlust. Ein Parley ist zu jedem Zeitpunkt das Ergebnis seiner bisher jüngsten
Verhandlung.

### A-5 — Nichts reist außerhalb des Parley

Jede Fähigkeit — ausdrücklich einschließlich der Byte-Kanäle über Seitenverbindungen — gehört dem
Parley: Sie erbt seine Identität und seine Autorisierung *konstruktiv* und endet mit ihm. Es gibt
keine Nebentür, die eigene Regeln hätte.

### A-6 — Fehler sind Verträge

Ein Fault trägt einen Code aus einem festen Katalog, eine für den Aufrufer bestimmte Botschaft und
eine Korrelations-Id. Unerwartete Fehler reisen als fester Satz plus Korrelations-Id; ihr Detail
gehört dem Betreiber, nicht dem Aufrufer.

### A-7 — Vorwärts heißt daneben

Ersetzen ist keine Operation, die das Protokoll kennt. Eine neue Protokollfassung, ein neues
Profile, ein neuer Contract tritt *neben* das Bestehende, und ein Acceptor MUST die aktuelle und
die vorigen Fassungen gleichzeitig bedienen — welchen Dialekt ein Parley spricht, entscheidet es
einzeln beim `hail`, wie ein TLS-Server, der mehrere Protokollversionen nebeneinander spricht.
Entfernen ist ein eigener, ausdrücklicher Schritt am Ende eines benannten Unterstützungsfensters,
niemals Nebenwirkung eines Updates. Das Fenster ist verhandelbar in seiner Länge, nicht in seinem
Minimum: Ein Acceptor MUST mindestens die aktuelle und die unmittelbar vorige Fassung bedienen —
*N und N−1*. Damit existiert immer ein bruchfreier Pfad: Der Server geht auf N, der Bestand
spricht weiter N−1 und zieht nach; erst dann darf N+1 kommen. Langsam, Sprosse für Sprosse — aber
es bricht nie. Wie viele weitere Fassungen eine Implementierung trägt, ist ihre Entscheidung; das
Minimum ist keine. Der Grund ist betrieblich: Flotten ziehen langsam nach, und App-Reviews testen
gegen den *laufenden* Server — der Server muss immer zuerst gehen können, und Zuerst-Gehen darf
den Bestand nicht brechen. Dieselbe Disziplin gilt für die SDK-Oberflächen: Neue API tritt neben
die alte; Entfernen ist ein angekündigter eigener Akt.

---

## § 4 Fähigkeitsmodell

Was ein Parley kann — jede Fähigkeit von Anfang an Teil des Protokolls, keine nachträglich
angebaut.

### Aufrufe, in beide Richtungen

Beide Peers bieten Contracts an, beide rufen auf. Ein Call korreliert über eine Id mit genau einem
Result oder Fault; ein Tell erwartet nichts. Methoden werden über `contract/method` plus
Argumentanzahl adressiert — die einzigen Diskriminatoren, die der Draht kennt. Zwei Methoden, die
unter demselben Namen und derselben Argumentanzahl erreichbar wären, sind ein Registrierungsfehler
beim Start, niemals eine stille Überschreibung.

**Jeder Call ist abbrechbar** — über seine Korrelations-Id, vom Moment des Absendens an. Das ist
Core-Semantik, kein Opt-in: Der Abbruch existiert, ohne dass die Methode dafür etwas deklarieren
müsste. **Im Handler wird er trotzdem sichtbar:** Das SDK reicht den Abbruch des laufenden Calls
in dem Idiom herein, das die Sprache dafür hat (§ 7) — als `CancellationToken`-Parameter, als
`context.Context`, als Coroutine- oder Task-Cancellation, als `AbortSignal`. Eigener Code
*innerhalb* einer Methode — eine Datenbankabfrage, eine Schleife, ein nachgelagerter Aufruf — kann
damit auf den Abbruch reagieren, statt ihn nur zu erleiden.

Abbruch ist kooperativ: Der Frame beendet verbindlich den *Call* — der Aufrufer erhält umgehend
einen Fault `cancelled` —, der Handler wird informiert und SHOULD seine Arbeit einstellen.
Abbruch-References (§ 2) ergänzen das für die Feinsteuerung: etwa einen einzelnen Stream beenden,
während der Call weiterläuft. Und das Ende des Parley bricht alles Laufende ab (I-10) —
Verbindungsverlust ist die Cancellation, die niemand senden muss.

### Asynchron von Grund auf

Es gibt keine synchronen Aufrufe. Auf dem Draht ist jede Interaktion ein Frame mit späterer
Antwort — Asynchronität ist keine Eigenschaft, die das Protokoll anbietet, sondern seine Natur.
SDK-Oberflächen folgen dem Nebenläufigkeits-Idiom ihrer Sprache (A-2): `async/await`, Futures,
Coroutinen — in Go das Blockieren einer Goroutine, was dort *das* asynchrone Idiom ist und keinen
Betriebssystem-Thread bindet. Was eine SDK-Oberfläche SHOULD NOT anbieten, ist das Blockieren von
Betriebssystem-Threads als Bequemlichkeit: Betriebserfahrung zeigt, dass solche Wrapper unter Last
Thread-Pools erschöpfen und der bequeme Pfad damit der gefährliche wird.

### Item-Streams

Jede Richtung kann Streams öffnen: als Rückgabe eines Calls, als Argument, oder freistehend. Ein
Stream hat eine Id, einen Besitzer, ein explizites Ende (vollständig, abgebrochen, fehlgeschlagen)
und Flusskontrolle als Verhandlungsgegenstand (Q-6).

### Byte-Kanäle

Große Nutzlasten — Dateien, Blobs, alles jenseits sinnvoller Frame-Größen — sind ein eigener
Kanaltyp mit Zuteilung, Besitzbindung, Kontingent und Frist. **Dass** es sie gibt, ist gesetzt;
**wie** sie transportiert werden, ist bewusst offen (Q-9). Der aussichtsreichste Kandidat ist eine
HTTP-Seitenverbindung: Kompression, Range- und Resume-Semantik, wohlverstandenes Streaming und
vorhandene Infrastruktur kommen geschenkt, die Frame-Verbindung bleibt latenzarm — und
Browser-Duplex-Streaming über den Socket ist bis heute unzuverlässig. Die Alternative ist
In-Band-Multiplexing durch die Frame-Verbindung: eine einzige Verbindung, keine zweite Tür, kein
Zuteilungs-Geheimnis unterwegs. Entscheidend ist, dass die Invarianten aus § 6 (I-1, I-2, I-4,
I-5) mechanismus-unabhängig formuliert sind — die Sicherheitszusagen hängen nicht an dieser Wahl.

Und damit die Mechanik-Frage offen bleiben kann, ohne dass Fähigkeiten von ihr abhängen, schuldet
ein Channel seine Eigenschaften *selbst* — egal, wie er reist:

- **Flusskontrolle** — ein langsamer Empfänger bremst den Sender; Puffer wachsen nicht unbegrenzt.
- **Vorrang der Frame-Verbindung** — ein laufender Transfer darf Calls nicht aushungern. Auf einer
  Seitenverbindung ist das geschenkt (eigene Röhre); in-band erzwingt es Stückelung und
  Verschränkung.
- **Wiederaufnahme** — ein abgerissener Transfer setzt an definierter Position fort, statt von
  vorn zu beginnen.
- **Kompression** — verhandelbar je Channel.
- **Integrität** — Grenzen und Prüfung beim Empfang, nicht per angekündigter Länge vertraut (I-5).

HTTP erfüllt diese Liste weitgehend nativ — Content-Encoding, Range, Flusskontrolle des Trägers —
das ist sein Vorsprung als Kandidat. Ein In-Band-Mechanismus muss sie im Protokoll nachbauen. Die
Liste ist der Maßstab, an dem Q-9 je Transportfamilie entschieden wird.

### References

Abbruch-Tokens, Streams und Channels treten als Argumente auf. Sie sind eigene, markierte
Wire-Typen. Ein Peer, der eine Reference erhält, kann sie bedienen (Token auslösen, Stream
konsumieren, Channel füllen) — und nur der Peer, dem sie gehört.

### Geleit (Conduct)

Credentials haben einen Lebenszyklus im Protokoll: Der Acceptor kann jederzeit eine *Challenge*
stellen; der Initiator antwortet mit einem frischen Nachweis, ohne dass die Verbindung fällt.
Prüfungen erfolgen gegen den Kontext, in dem die Verbindung ankam. Ablauf und Entzug wirken auf
die laufende Verbindung — bis zum serverseitig ausgesprochenen Abbruch. Autorisierung ist pro
Contract, pro Methode und pro Channel deklarierbar.

### Adressierung jenseits der Einzelverbindung

Der Acceptor führt das **Muster** (§ 2): je laufendem Parley die Sitzungsidentität, den
beglaubigten Principal, freie Metadaten als Schlüssel/Wert-Paare und die Gruppenzugehörigkeiten.
Adressiert wird nicht durch Aufzählen von Verbindungen, sondern durch **Auswahl über das Muster**:
Ein Auswahlausdruck — Gruppe, Metadatum, Principal, kombinierbar — bestimmt die Empfängermenge, an
die ein Call, Tell oder Stream geht. *„Alle in Gruppe xy mit Metadatum legacy"* ist ein Ausdruck,
kein Schleifencode. Dieselbe Auswahl ist auch abfragbar, ohne zu senden — wer ist da, wie viele.

Auswahl kennt drei Stufen, mit ehrlicher Reichweite:

- **Deklarative Ausdrücke** — Gruppe, Metadatum, Principal-Merkmale als Daten. Reisen zwischen
  Knoten, werden flottenweit ausgewertet (A-3 in weiterer Gestalt: Code kann den Draht nicht
  überqueren, Daten schon).
- **Benannte Prädikate** — in der Anwendung registrierte Auswahllogik, adressiert als Name plus
  Parameter. Der Name reist als Datum; jeder Knoten wertet das Prädikat lokal gegen seine eigenen
  Parleys aus und stellt an seine Treffer zu (Scatter-Gather). Flottenweit, weil eine homogene
  Flotte denselben Code trägt — in gemischten Clustern (Q-5, Stufe b) nur, wenn alle
  Implementierungen denselben Namen registrieren.
- **Anonyme lokale Prädikate** — beliebiger Code über die eigenen Objekte eines Knotens. Zulässig,
  aber die Reichweite MUST als knoten-lokal ausgewiesen sein.

Und eine Besitzregel: **Das Muster gehört dem Acceptor.** Metadaten setzt die Server-Seite; was
der Initiator liefert, sind Anmeldedaten und Wünsche, niemals Selbstbeschreibung mit Wirkung — ein
Client, der sich selbst taggt, wäre ein Client, der seine eigene Autorisierung schreibt.

Gruppenmitgliedschaft ist Sitzungszustand mit definierten Regeln für Wiederverbindung.

**Der Acceptor ist logisch, nicht physisch.** Hinter der Rolle kann eine Flotte von Knoten stehen;
Mehrknoten-Betrieb ist von Anfang an gesetzt, keine spätere Erweiterung. Adressierung — einzeln,
Gruppe, gesamt — wirkt über die gesamte Flotte, und ein Initiator MUST NOT unterscheiden können,
mit wie vielen Knoten er es zu tun hat: Die Verteilung zwischen Knoten (die *Backplane*) ist
unsichtbar, taucht in keiner Verhandlung auf und ist kein Profile. Welche Träger sie nutzt —
relationale Datenbank, Key-Value-Store, Broker — ist Sache der Server-Implementierung und
steckbar. Welche Zustellzusagen Gruppen-Sendungen über Knoten hinweg haben (Ordnung, Aufholen beim
Knotenbeitritt), und wie viel davon Spezifikation wird, ist Q-5.

### Beobachtbarkeit

Jeder Call und jedes Tell kann W3C-Trace-Context tragen (`traceparent`/`tracestate`), sodass ein
Trace vom Aufrufer über den Acceptor bis in den Handler des Gegenübers durchläuft. Optional und
additiv: Empfänger ohne Tracing ignorieren die Felder.

### Arbeits-Taxonomie der Frames

| Frame | Richtung | Zweck |
|---|---|---|
| `hail` / `terms` | I → A / A → I | Verbindungsaufnahme und Verhandlungsergebnis: Protokollfassung, Format, Profile, Selbstauskunft. Ausgang: Annahme, Umleitung oder Ablehnung (§ 5) |
| `call` / `result` / `fault` | beide | Aufruf mit Korrelation; genau ein Abschluss pro Call |
| `tell` | beide | Fire-and-forget-Nachricht an eine Methode |
| `stream-open` / `item` / `end` | beide | Item-Streams mit explizitem Abschlussgrund |
| `channel-grant` | A → I | Zuteilung eines Byte-Kanals: Weg oder Id (je nach Q-9), Frist, Bindung |
| `challenge` / `proof` | A → I / I → A | Credential-Herausforderung und frischer Nachweis |
| `cancel` | beide | Abbruch eines laufenden Calls über seine Id — oder Auslösen einer Abbruch-Reference |
| `pulse` | beide | Lebenszeichen; Grundlage der Timeout-Verhandlung |
| `farewell` | beide | Geordneter Abschied mit Grund |

Arbeitsnamen zur Diskussion der Semantik — die endgültigen Frame-Namen und das Wire-Format sind
Gegenstand der Spezifikation, nicht dieses Dokuments.

---

## § 5 Transportmodell

Der Grund, dieses Protokoll zu benutzen statt eines rohen Sockets: Es nimmt das Beste, was Client
und Netz hergeben — und fällt geordnet zurück, wenn das Beste nicht geht.

Der Aufbau hat zwei Schichten, und sie zu trennen ist die Voraussetzung echter
Transportunabhängigkeit. **Schicht eins stellt die Röhre her** — wie, ist Sache der
Transportfamilie. **Schicht zwei verhandelt die Sitzung** — Protokollfassung, Format, Profile,
Geleit — und reist *in-band*, als die ersten Frames auf der Röhre selbst (`hail`/`terms`, § 4). So
verhandeln auch TLS, SSH und AMQP: Die Sitzungsverhandlung setzt kein zweites Protokoll voraus,
nur eine Röhre.

Ein Transport schuldet dem Protokoll genau drei Dinge: geordnete Zustellung, vollständige Frames,
erkennbares Verbindungsende. Alles andere — Korrelation, Streams, Geleit — lebt oberhalb und ist
transportunabhängig.

### Transportfamilien

Die **Web-Familie** ist das Flaggschiff: **WebSocket** (bidirektional, bevorzugt), **Server-Sent
Events + HTTP** (Abwärtskanal als Ereignisstrom, Aufwärtskanal als Requests) und **Long Polling**
(beide Richtungen über gehaltene Requests). Nur diese Familie hat eine Vorab-Auswahl über HTTP —
nicht als Grundprinzip, sondern weil im Browser die Röhre selbst erst gefunden werden muss: Ob
WebSocket durchkommt, weiß man nicht, bevor man es versucht hat, und die Fallback-Kette überlebt
Corporate-Proxies, restriktive Gateways und Mobilfunknetze. Die Auswahl gehört der Familie; die
Sitzungsverhandlung danach ist dieselbe wie überall.

Andere Familien brauchen keine Vorab-Auswahl und damit kein HTTP: **IPC** (Named Pipes, Unix
Domain Sockets) und **TCP/TLS** für Dienste auf derselben Maschine oder im selben Netz,
perspektivisch **QUIC/WebTransport** — und **getunnelt**: jeder bidirektionale Stream eines
anderen Systems, der die drei Pflichten erfüllt, kann Parley-Frames tragen. Röhre verbinden,
`hail` senden — mehr Aufbau gibt es dort nicht. Neue Familien sind Erweiterungspunkte, keine
Spezifikationsänderungen.

### Ungleiche Stände

Flotten hinken: Mobile Clients durchlaufen Store-Reviews und Nutzer, die nicht aktualisieren — ein
neuer Acceptor spricht in der Praxis *immer* auch mit alten Initiators. Das regierende Prinzip
dafür ist A-7: Vorwärts heißt daneben, der Server geht zuerst, und Zuerst-Gehen bricht den Bestand
nicht. Die Werkzeuge, mit denen A-7 ausgeführt wird:

- **Selbstauskunft im `hail`** — Implementierung, Anwendungsversion, Plattform. Die Server-Seite
  stempelt sie als Metadaten ins Muster (§ 4): Damit ist *„alle mit Version < 3.0"* eine
  gewöhnliche Auswahl — für Update-Hinweise, degradierte Bedienung oder schlicht Sichtbarkeit, wie
  viele Nachzügler es noch gibt. Selbstauskunft ist Behauptung, nicht Nachweis — sie steuert
  Komfort, niemals Autorisierung (§ 4).
- **Nebeneinander** — der Alltagsweg, und er braucht kein einziges Protokoll-Feature: Contracts
  sind benannte Mengen (§ 2), ein Acceptor beherbergt beliebig viele. Der v2-Contract wird unter
  eigenem Wire-Namen neben dem alten registriert (`chat` und `chat.v2`); alte Initiators rufen
  weiter den alten, neue den neuen — der Client wählt durch das, was er aufruft. Sind alle
  nachgezogen, wird der alte Contract schlicht ausgetragen. Wer Versionen lieber als getrennte
  Endpunkte führt (zwei Pfade, zwei Pipe-Namen), kann das auf Familienebene ebenso.
- **Umleitung** — die `terms` können statt Annahme ein Ziel nennen: *„geh dorthin".* Das schwerere
  Geschütz, wenn die alte Server-Version als eigenes Deployment unangetastet weiterlaufen soll,
  während der neue Acceptor alte Initiators zu ihr routet — Legacy wird betrieben, nicht portiert,
  und irgendwann abgeschaltet. SDKs folgen automatisch; die Spezifikation begrenzt
  Umleitungsketten, und Credentials folgen einer Umleitung MUST nur unter definierten
  Vertrauensbedingungen, nie blind an beliebige Ziele.
- **Definierte Ablehnung** — ein Fault (Arbeitsname `upgrade_required`), den der Acceptor bei den
  `terms` oder später aussprechen kann und den SDKs unterscheidbar durchreichen: Die App zeigt
  „bitte aktualisieren" statt eines generischen Verbindungsfehlers. Der natürliche Lebenszyklus:
  heute umleiten, nach Abschalten des Legacy-Ziels ablehnen.
- **Fähigkeits-Kompatibilität über Profile** — ein alter Initiator meldet ein neues Profile
  schlicht nicht an; die Verhandlung schneidet das Gespräch auf die gemeinsame Menge zu (§ 8).
  Neues ist opt-in, nie stillschweigende Anforderung.
- **Contract-Kompatibilität** — die Regeln, nach denen Contracts wachsen dürfen, ohne alte
  Gegenüber zu brechen, sind Q-12.

Nach Verbindungsverlust verhandelt der Initiator neu. Ob eine Sitzung dabei *fortgesetzt* werden
kann — gleiche Identität, gepufferte Zustellung — ist ein eigenes Profile (§ 8) mit eigener
Verhandlung; die Basisgarantie ist der saubere Neuaufbau.

---

## § 6 Invarianten — by design unmöglich

Betriebserfahrung, destilliert zu Sätzen, die keine Implementierung verletzen kann, ohne die
Konformität zu verlieren. Jede dieser Invarianten beantwortet eine real beobachtete Fehlerklasse.

- **I-1 — Kein Kanal ohne Identität.** Ein Channel existiert nur als Zuteilung an ein bestehendes
  Parley und ist an dessen Identität gebunden — gleichgültig, ob er in-band oder über eine
  Seitenverbindung geführt wird. Eine Implementierung, in der ein Transferweg eigene
  Autorisierungsregeln haben *könnte*, ist nicht konform — die Erbschaft ist konstruktiv, nicht
  konfigurierbar.
- **I-2 — Fremder Besitz ist unerreichbar.** Streams, Channels und Abbruch-Tokens gehören dem
  Parley, das sie erzeugt hat. Ein Frame, der fremden Besitz adressiert, wird abgewiesen — und
  zwar ununterscheidbar von „existiert nicht", damit die Abweisung nicht als Sonde taugt.
- **I-3 — References werden erklärt, nie erraten.** Ob ein Argument Wert oder Reference ist, sagt
  der Frame. Eine Implementierung, die Referenzen an der Form von Nutzdaten erkennt, ist nicht
  konform.
- **I-4 — Erwerb ist kontingentiert.** Alles, was ein Peer beim Gegenüber belegt — Channels,
  offene Streams, ausstehende Calls — hat ein verhandelbares Kontingent mit definiertem Fehlercode
  und verfällt spätestens mit dem Parley.
- **I-5 — Prüfen vor Puffern.** Kein Peer nimmt Nutzlast entgegen, bevor er ihre Zuteilung geprüft
  hat, und keine Übernahme vertraut einer angekündigten Länge: Grenzen werden beim Lesen
  durchgesetzt.
- **I-6 — Interna überqueren den Draht nicht.** Ein unerwarteter Fehler reist als fester Satz plus
  Korrelations-Id. Typnamen, Pfade, Verbindungsdetails und Ursachenketten erscheinen
  ausschließlich im Log des Betreibers, unter derselben Id.
- **I-7 — Geleit prüft im Ankunftskontext.** Credential-Prüfungen sehen den Kontext, in dem die
  Verbindung ankam — Host, Schema, gestempelte Metadaten. Ein Nachweis kann eine laufende Sitzung
  niemals in eine *schwächere* Prüfstufe befördern.
- **I-8 — Kollisionen sind Startfehler.** Zwei Methoden unter demselben Wire-Namen und derselben
  Argumentanzahl verhindern den Start. Es gibt keinen Zustand, in dem „der Letzte gewinnt".
- **I-9 — Kein Speicher wächst am Gegenüber.** Caches, die von empfangenen Bezeichnern gespeist
  werden, sind begrenzt, und Fehlschläge werden nicht dauerhaft gemerkt. Ein Peer kann das
  Gedächtnis seines Gegenübers nicht wachsen lassen.
- **I-10 — Abschied räumt auf.** Das Ende eines Parley beendet alles, was ihm gehört: Streams
  werden geschlossen, Channels verfallen, Tokens lösen aus, Wartende erfahren es. Nichts überlebt
  seinen Besitzer.

---

## § 7 Die Sprachschnittmenge

Sechs Zielsprachen definieren, was Semantik werden darf (A-2). Eine Zeile qualifiziert sich nur
mit sechs gefüllten Zellen — die Tabelle ist das Werkzeug, mit dem jeder künftige Vorschlag
geprüft wird.

| Semantik | C# | TypeScript | Rust | Swift | Kotlin | Go |
|---|---|---|---|---|---|---|
| Typisierte Proxies | Source Generator | ES `Proxy` + Generics | Proc-Macro | Macro | KSP | Codegen |
| Handler-Registrierung | Attribute | String + Mapped Types | Macro | Macro | KSP/DSL | Codegen |
| Item-Streams | `IAsyncEnumerable` | Async-Iterator | `Stream` | `AsyncSequence` | `Flow` | Channel |
| Abbruch | `CancellationToken` | `AbortSignal` | Token (tokio) | Task-Cancellation | Coroutine-Job | `context.Context` |
| Byte-Kanäle | `Stream` | `ReadableStream`/Blob | `AsyncRead` | `AsyncBytes` | `Flow<ByteString>`/okio | `io.Reader` |
| Tracing | `Activity` | OTel JS | `tracing` | OTel Swift | OTel Kotlin | OTel Go |

**Zur TypeScript-Zelle:** Interfaces existieren dort nur zur Compile-Zeit — gebraucht werden sie
zur Laufzeit aber auch nicht. Ein ES-`Proxy` fängt Methodenzugriffe ab; der Property-Name ist zur
Laufzeit ohnehin ein String und wird zum Wire-Namen, während der Generic die Aufrufe vollständig
typisiert. Einzige Auflage an die Spezifikation: explizites Namens-Mapping als Ausweich für
Umgebungen, die Property-Namen minifizieren.

---

## § 8 Konformität und Profile

„Parley sprechen" ist ein Testergebnis, kein Anspruch. Und nicht jede Implementierung muss alles
können — aber jede muss wahrhaftig anmelden, was sie kann.

Die Konformitätssuite ist Bestandteil der Spezifikation und läuft gegen jede Implementierung —
Initiator wie Acceptor. Fähigkeiten sind in **Profile** gebündelt, die in der Verhandlung
angemeldet werden:

- **Core** — Verhandlung, Calls/Tells beidseitig, Faults, Abbruch, Geleit, Pulse. Ohne Core kein
  Parley.
- **Streams** — Item-Streams in beide Richtungen samt Flusskontrolle.
- **Channels** — Byte-Kanäle mit Zuteilung, Bindung und Kontingent.
- **Resume** — Sitzungsfortsetzung nach Verbindungsverlust mit gepufferter Zustellung.
- **Trace** — Durchreichung des W3C-Trace-Contexts.

Ein Peer, der ein Profile nicht angemeldet hat, bekommt dessen Frames nie zu sehen — die
Verhandlung schneidet das Gespräch auf die gemeinsame Menge zu. Genau dadurch bleibt die
Schnittmengen-Regel auch nach Erscheinen neuer Profile wahr: Neues ist opt-in per Verhandlung, nie
stillschweigende Anforderung.

---

## § 9 Nicht-Ziele

- **Kein Medien-Streaming, kein Peer-to-Peer.** Audio, Video und Datenkanäle zwischen Endgeräten
  sind das Feld von WebRTC. Parley verbindet Clients mit Diensten.
- **Kein Message-Bus.** Parley kennt keine Topics, keine persistenten Queues, keine
  Zustellgarantien über Sitzungsgrenzen hinweg (Resume ausgenommen). Wer einen Broker braucht,
  braucht einen Broker.
- **Kein Request/Response-Ersatz.** Eine REST- oder RPC-API für anonyme, zustandslose
  Einzelaufrufe bleibt das einfachere Werkzeug. Parley beginnt dort, wo eine *Sitzung* mit
  Rückkanal gebraucht wird.
- **Keine Geschäftslogik im Protokoll.** Presence-Semantik, Räume mit Rechtemodellen,
  Kollaborations-Datentypen sind Anwendungen *auf* Parley, nicht Teile davon.

---

## § 10 Offene Entwurfsfragen

Die Gabelungen, die vor der Spezifikation zu entscheiden sind — gereiht nach Tragweite.

- **Q-1 — Wo lebt der Contract?** Neutrale Contract-Datei mit Codegen pro Sprache (das
  Schema-Modell) — oder Source-of-Truth pro Sprache mit Wire-Namensdisziplin? Go erzwingt die
  Frage, weil es ohne externen Codegen nicht auskommt; Sprachen mit Macros könnten beides. Die
  Antwort bestimmt Tooling, Versionierung und Einstiegshürde mehr als jede andere.
- **Q-2 — Welche Serialisierungen sind normativ?** JSON als Pflichtformat ist gesetzt. Das binäre
  Zweitformat — MessagePack, CBOR, oder keins in Fassung 1 — ist offen, ebenso ob das Format pro
  Parley oder pro Frame verhandelt wird.
- **Q-3 — Wie altert das Protokoll?** Reicht die Profile-Verhandlung als Evolutionsmechanik, oder
  braucht es zusätzlich eine Protokollfassung im `hail`? Vorschlag zur Prüfung: Fassung für
  inkompatible Frame-Änderungen, Profile für alles Additive.
- **Q-4 — Wie viel Resume in Fassung 1?** Sitzungsfortsetzung mit Puffern und Replay ist das
  teuerste Profile und das einzige mit Speicherhaltung beim Acceptor. Fassung 1 ohne Resume
  shippen — oder ist die Fortsetzung so zentral, dass sie von Anfang an spezifiziert gehört, damit
  sie nicht angebaut wird wie das, was dieses Dokument zu vermeiden gelobt?
- **Q-5 — Wie viel Backplane wird Spezifikation?** Dass der Acceptor eine Flotte sein kann, ist
  gesetzt (§ 4) — offen ist, wie viel davon normativ wird. Stufe (a): Die Spezifikation definiert
  nur die *sichtbare* Semantik (Gruppen-Zustellzusagen, Ordnung, Aufholen beim Knotenbeitritt),
  jede Server-Implementierung verteilt intern, wie sie will — mit steckbaren Trägern hinter einer
  eigenen Provider-Schnittstelle. Stufe (b): Zusätzlich wird das Knoten-zu-Knoten-Protokoll
  spezifiziert — dann könnten Knoten *verschiedener* Implementierungen einen Cluster bilden, etwa
  gemischte Sprachen hinter einem Load-Balancer. Stufe (b) ist deutlich teurer und bindet früh;
  sie lässt sich aber als spätere Ergänzung nachrüsten, solange (a) die sichtbare Semantik exakt
  festlegt. Innerhalb von (a) bleibt auch die **Zustellstrategie** Trägersache, mit zwei
  Archetypen. Das **Verzeichnis-Modell**: Das Muster liegt als abfragbare Tabelle im gemeinsamen
  Träger — eine Zeile je Parley, samt Heimatknoten. Der sendende Knoten wertet die Auswahl einmal
  dort aus und schickt an jeden betroffenen Knoten die Nachricht plus die Ziel-Ids; der
  Heimatknoten stellt zu. Natürlich für relationale und Key-Value-Träger, macht Zählen und
  Blättern (Q-10) billig — braucht aber Lebenszeichen-Pflege, denn ein abgestürzter Knoten
  hinterlässt Geisterzeilen. Das **Fan-out-Modell**: Die Sende-Absicht geht an alle Knoten, jeder
  filtert lokal gegen seine eigenen Parleys. Natürlich für Broker-Träger, kennt keine
  Geisterzeilen, macht reiche Abfragen aber teuer. Beide sind von außen ununterscheidbar — bis auf
  eine Semantik, die die Spezifikation deshalb modellunabhängig festlegen muss: **Auswahl wirkt
  auf einen Schnappschuss.** Zwischen Auswertung und Zustellung kann ein Parley enden oder
  umziehen; Zustellung an eine ermittelte Id ist Best-Effort, eine nicht mehr vorhandene Id wird
  still übersprungen.
- **Q-6 — Flusskontrolle für Streams.** Credit-basiert, fensterbasiert, oder in Fassung 1 dem
  Transport überlassen? Die Antwort entscheidet, ob ein langsamer Konsument den Erzeuger bremst
  oder Puffer wachsen.
- **Q-7 — Wie heißen die Dinge endgültig?** Die Arbeitsnamen aus § 4 sind Vorschlag: `hail`,
  `terms`, `conduct`, `farewell` tragen die Parley-Metapher in die Frame-Sprache. Zu entscheiden:
  Metapher-Namen oder neutrale (`open`, `accept`, `auth`, `close`) — Charakter gegen
  Selbsterklärung.
- **Q-8 — Was verlangt Konformität mindestens an Transports?** Muss ein konformer Acceptor die
  volle Web-Familie bedienen, oder ist WebSocket Pflicht und der Rest Profile? Die Fallback-Kette
  ist das Wertversprechen — aber für einen internen Dienst hinter eigener Infrastruktur ist sie
  möglicherweise totes Gewicht. In Familien gedacht schärfer: Ist die Web-Familie Pflicht für
  jeden Acceptor, oder gibt es konforme reine IPC- oder TCP-Acceptors — etwa interne Dienste, die
  nie einen Browser sehen?
- **Q-9 — Wie reisen Byte-Kanäle?** HTTP-Seitenverbindung (Kompression, Range/Resume, vorhandene
  Infrastruktur — der Favorit) oder In-Band-Multiplexing durch die Frame-Verbindung (eine
  Verbindung, keine zweite Tür)? Auch denkbar: beides als getrennte Profile mit Verhandlung. Die
  Invarianten I-1, I-2, I-4 und I-5 gelten so oder so — zu entscheiden ist Mechanik, nicht
  Sicherheit. Das Prüfkriterium für den Favoriten: **Wie weist die Seitenverbindung die
  Parley-Identität nach?** Die Zuteilung muss das Geleit erben (A-5, I-1) — eröffnet der
  Mechanismus eine zweite, eigene Auth-Welt, fällt er durch. Und eine Abhängigkeit zu § 5: Ein
  reiner IPC- oder TCP-Acceptor hat kein HTTP — wäre die Seitenverbindung der einzige Mechanismus,
  gäbe es dort kein Channels-Profile. Zugleich entfallen auf lokalen Röhren die Browser-Grenzen,
  die gegen In-Band sprachen. Das legt nahe, den Mechanismus **je Transportfamilie** festzulegen:
  Web-Familie über die Seitenverbindung, IPC/TCP in-band. Der Maßstab für beide ist die
  Pflichtenliste in § 4 — HTTP bringt sie mit, In-Band muss sie nachbauen, allen voran den Vorrang
  der Frame-Verbindung.
- **Q-10 — Wie ausdrucksstark ist die Muster-Auswahl?** Welche Dimensionen kennt der
  Auswahlausdruck — Gruppe, Metadatum (Existenz und Gleichheit), Principal-Merkmale? Nur
  Konjunktion, oder auch Disjunktion und Negation? Gibt es Zählen und Blättern über große
  Empfängermengen? Jede Dimension muss flottenweit auswertbar bleiben (§ 4) — die Grenze verläuft
  dort, wo aus einem Datenausdruck eine Abfragesprache wird, die jeder Träger der Backplane erst
  implementieren muss.
- **Q-11 — Was wird aus generischen Contract-Methoden?** In einer Welt, in der beide Seiten
  geschlossen sind (A-3), kann keine Seite zur Laufzeit instanziieren. Der naheliegende Weg:
  Generik ist SDK-Oberfläche, die zu eigenen Wire-Namen je deklarierter Instanziierung expandiert
  — der Contract auf dem Draht kennt dann keine Typparameter. Zu entscheiden: ob das
  Contract-Modell Typparameter überhaupt kennt, oder ob die Expansion das Modell *ist*.
- **Q-12 — Wie altern Contracts?** Q-3 regelt die Evolution des Protokolls — offen ist die der
  *Contracts*. Adressierung über Name plus Argumentanzahl macht das Hinzufügen eines Parameters
  zum Bruch. Welche Kompatibilitätsregeln gelten: additive Methoden, optionale Parameter mit
  definierter Befüllung, Deprecation, Umbenennung über Wire-Namen? Mit Codegen (Q-1) stellt sich
  die Frage als Schema-Evolution. Für Teams, die ohne Stichtag ausliefern wollen, ist dies die
  adoption-entscheidende Frage.
- **Q-13 — Reihenfolge und Nebenläufigkeit der Zustellung.** Die Röhre liefert Frames geordnet —
  aber führt der Empfänger Handler *pro Parley* seriell aus oder parallel? Zwei Tells in
  Reihenfolge A, B: darf B abgeschlossen sein, bevor A beginnt? Das ist beobachtbare Semantik und
  muss festgelegt werden, sonst weichen Implementierungen zwangsläufig voneinander ab. Optionen:
  strikt seriell je Parley (einfach, aber ein langsamer Handler staut alles), parallel mit
  verhandelter Obergrenze, oder deklarierbar je Methode.

---

*Dieses Dokument destilliert mehrjährige Betriebserfahrung mit einem Vorgänger-Framework
desselben Stewards; es steht absichtlich ohne Verweise auf dessen Technik, damit jede Festlegung
sich hier begründen muss statt dort. Grundlagenfassung 0.1 — zur Diskussion, nicht zur
Implementierung.*
