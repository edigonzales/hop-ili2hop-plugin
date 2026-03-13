# Anforderungsdokument — INTERLIS-Input-Transformer als Apache-Hop-Plugin

> Arbeitsstand: fachlicher und technischer Anforderungsentwurf auf Basis der beschriebenen Zielsetzung, der vorgeschlagenen Plugin-Architektur sowie des Vergleichs mit ili2fme/FME.

> **Zweck dieses Dokuments**  
> Dieses Dokument beschreibt die Ziele, Anforderungen, Architekturprinzipien, offenen Punkte und Abgrenzungen für ein Apache-Hop-Plugin, das INTERLIS-Daten einliest und in Hop-taugliche Row-Streams überführt. Es ist bewusst so formuliert, dass es als Grundlage für Konzeption, Implementierung, interne Abstimmung oder spätere Ausschreibung weiterverwendet werden kann.

## 1. Ausgangslage und Zielbild

Ein Apache-Hop-Plugin soll INTERLIS-Daten mit Hilfe vorhandener INTERLIS-Bibliotheken und Parser (z. B. ili2c, iox-ili und verwandte Bausteine) einlesen und in einer für Hop verarbeitbaren Form bereitstellen.

Die zentrale Herausforderung liegt nicht im Parsen an sich, sondern in der Abbildung von INTERLIS-Konstrukten auf Hop-Row-Streams. Besonders kritisch sind vernetzte Daten über Assoziationen sowie verschachtelte Daten über Strukturen und wiederholte Strukturwerte.

Das Ziel ist ein robustes, modellgetriebenes Input-Plugin, das einfache INTERLIS-Modelle effizient verarbeitet, zugleich aber auch komplexere INTERLIS-Konstrukte nachvollziehbar und kontrolliert in Hop transformierbar macht.

## 2. Problemkern

Apache Hop arbeitet im Kern mit tabellarisch orientierten Row-Streams. INTERLIS beschreibt dagegen objektartige, typisierte, vernetzte und teils verschachtelte Datenstrukturen.

Eine direkte 1:1-Abbildung „ein INTERLIS-Objekt = eine Hop-Row“ ist für einfache Klassen denkbar, bricht jedoch bei mehrfachen Geometrien, Assoziationen, LIST/BAG OF STRUCTURE, tiefer Verschachtelung, Vererbung und komplexen Referenzmustern schnell zusammen.

Das Plugin muss daher ein bewusstes Zwischenmodell bereitstellen, das einerseits INTERLIS-Semantik hinreichend erhält und andererseits mit den Verarbeitungseigenschaften von Hop kompatibel bleibt.

## 3. Erkenntnisse aus dem Vergleich mit ili2fme/FME

Das bestehende Projekt ili2fme zeigt, dass die konzeptionelle Herausforderung bereits in einem anderen ETL-Umfeld gelöst wurde. Der Vergleich ist fachlich nützlich, die technische Zielplattform ist jedoch anders.

FME verfügt über ein reichhaltigeres Feature-Modell als Apache Hop. FME-Features können neben Attributen auch native Geometrie, Listenattribute und format-spezifische Metadaten bequem mitführen. Dadurch kann ili2fme komplexe INTERLIS-Inhalte häufiger innerhalb eines einzigen FME-Features repräsentieren.

ili2fme verwendet insbesondere qualifizierte INTERLIS-Namen für Feature-Types, explizite Metaattribute wie xtf_id, xtf_basket, xtf_class und xtf_operation, spezielle Feature-Types für Transfer- und Basket-Metadaten, konfigurierbare Vererbungsstrategien sowie FME-Listenattribute für BAG/LIST OF STRUCTURE.

Diese Lösung ist für FME passend, weil FME Listen- und Geometrie-Konzepte nativ unterstützt. Apache Hop besitzt kein gleichwertiges Standard-Featureobjekt. Daher ist eine 1:1-Übernahme des ili2fme-Ansatzes nicht angezeigt.

Aus ili2fme lassen sich dennoch tragfähige Prinzipien ableiten: Typinformationen explizit mitführen, Metadaten nicht verstecken, Schema aus dem Modell ableiten, Hybrid-Strategien für schwierige Typen zulassen und komplexe Konstrukte nicht gewaltsam vollständig flatten.

## 4. Zielsystem und Zielobjekte

Das zu entwickelnde Plugin ist ein externer Apache-Hop-Transform-Typ zum Einlesen von INTERLIS-Daten.

Das Plugin soll mindestens INTERLIS-Modelle laden, Datenbestände lesen, das Zielschema modellgetrieben ableiten und die Daten in definierte Hop-Row-Streams emittieren.

Das Plugin soll nicht bloß eine Dateilesekomponente sein, sondern eine kontrollierte Abbildungs- und Serialisierungskomponente zwischen INTERLIS und Hop darstellen.

## 5. Fachliche Anforderungen

### 5.1 Muss-Anforderungen

| ID | Anforderung | Priorität | Erläuterung |
| --- | --- | --- | --- |
| F-01 | Das Plugin muss INTERLIS-Modelle und INTERLIS-Daten mit bestehenden INTERLIS-Java-Bausteinen verarbeiten können. | Muss | Das Parsen selbst soll auf etablierten Komponenten aufbauen statt neu implementiert zu werden. |
| F-02 | Das Plugin muss das Ausgabeschema vor der Datenverarbeitung aus dem INTERLIS-Modell ableiten können. | Muss | Das Schema darf nicht erst opportunistisch während des Lesens „erraten“ werden. |
| F-03 | Das Plugin muss einfache Objektklassen als Haupt-Row-Stream ausgeben können. | Muss | Basisfunktionalität für einfache ETL-Strecken. |
| F-04 | Das Plugin muss Referenzen und Assoziationen in einer Form ausgeben können, die downstream eindeutig auswertbar ist. | Muss | Mindestens über OID-/Ref-Felder oder separate Link-Streams. |
| F-05 | Das Plugin muss verschachtelte Strukturen und wiederholte Strukturwerte abbilden können. | Muss | Insbesondere LIST/BAG OF STRUCTURE dürfen nicht stillschweigend verloren gehen. |
| F-06 | Das Plugin muss technische Metadaten wie OID, Basket, Topic, Klasse und Quelle explizit bereitstellen können. | Muss | Diese Metadaten sind für Rückverfolgbarkeit und Rekonstruktion wesentlich. |
| F-07 | Das Plugin muss Fehlerfälle kontrolliert behandeln und einen nachvollziehbaren Fehlerkanal oder Fehler-Output vorsehen. | Muss | Fehler dürfen nicht nur im Log verschwinden. |
| F-08 | Das Plugin muss streaming-orientiert arbeiten und darf komplexe Daten nicht unnötig vollständig im Speicher materialisieren. | Muss | Relevant für größere Datenmengen und stabile Laufzeiten. |

### 5.2 Soll-Anforderungen

| ID | Anforderung | Priorität | Erläuterung |
| --- | --- | --- | --- |
| F-09 | Das Plugin soll mehrere Ausgabemodi unterstützen. | Soll | Mindestens Flat Mode, Relational Mode und Hybrid/Raw Mode. |
| F-10 | Das Plugin soll unterschiedliche Strategien für Vererbung unterstützen. | Soll | Analog zu Superclass- und Subclass-Strategie. |
| F-11 | Das Plugin soll unterschiedliche Strategien für Geometrien unterstützen. | Soll | Mindestens WKT als Default, optional WKB/Binary oder alternative Ausgabeformen. |
| F-12 | Das Plugin soll Enumerationen konfigurierbar als Code, Text oder beide Formen ausgeben können. | Soll | Erleichtert die Anbindung unterschiedlicher Zielsysteme. |
| F-13 | Das Plugin soll technische Metadaten optional auch in separaten Meta-Streams bereitstellen können. | Soll | Analog zur Idee von Transfer-/Basket-/Delete-Entitäten. |
| F-14 | Das Plugin soll problematische oder seltene Konstrukte alternativ serialisiert ausgeben können. | Soll | Zum Beispiel als JSON/XML-ähnlicher Inhalt in Feldern. |

## 6. Nichtfunktionale Anforderungen

| ID | Kriterium | Anforderung |
| --- | --- | --- |
| NF-01 | Nachvollziehbarkeit | Die gewählte Abbildung muss für Entwickler und Pipeline-Autoren verständlich und dokumentierbar sein. |
| NF-02 | Determinismus | Gleiche Modelle und gleiche Daten sollen zu reproduzierbaren Schemas und Row-Strukturen führen. |
| NF-03 | Erweiterbarkeit | Die Architektur soll weitere INTERLIS-Konstrukte, zusätzliche Ausgabemodi und künftige Optimierungen ohne Grundumbau zulassen. |
| NF-04 | Testbarkeit | Modellanalyse, Mapping und Row-Emission sollen voneinander getrennt sein, damit sie isoliert getestet werden können. |
| NF-05 | Bedienbarkeit | Die Transform-Konfiguration in Hop soll fachlich verständlich sein und die wichtigen Mapping-Optionen explizit machen. |
| NF-06 | Leistung | Das Plugin soll auch für größere Bestände praktikabel bleiben; Modellkompilierung und Mapping sollen nach Möglichkeit wiederverwendbar sein. |
| NF-07 | Lizenzverträglichkeit | Abhängigkeiten mit abweichender Lizenzlage sind zu berücksichtigen; das Plugin soll als externes Plugin konzipiert werden. |

## 7. Empfohlene fachliche Abbildungslogik

Die empfohlene Grundentscheidung lautet: Komplexe INTERLIS-Daten sollen nicht pauschal in eine einzige Megarow gezwungen werden. Stattdessen soll das Plugin ein kanonisches Zwischenmodell erzeugen, das als mehrere logisch getrennte Entitäten beziehungsweise Streams ausgeprägt werden kann.

Die bevorzugte Standardstrategie ist relational-hybrid: einfache Klassenattribute direkt im Haupt-Stream, Referenzen und Assoziationen kontrolliert als OID/FK-Felder und/oder Link-Streams, komplexe oder wiederholte Strukturen als Child-Streams, problematische Spezialfälle optional zusätzlich serialisiert.

### 7.1 Ausgabemodi

| Modus | Beschreibung | Eignung |
| --- | --- | --- |
| Flat Mode | Eine Haupt-Row pro INTERLIS-Objekt; nur für einfache Modelle oder begrenzte Tiefe geeignet. | Einfach, aber nur eingeschränkt robust. |
| Relational Mode | Hauptdaten, Strukturen, Assoziationen und Metadaten werden in getrennten Streams/Entitäten ausgegeben. | Bevorzugter Produktionsmodus für komplexe Modelle. |
| Hybrid/Raw Mode | Hauptdaten werden flach ausgegeben; komplexe Inhalte werden zusätzlich oder alternativ serialisiert mitgegeben. | Praktischer Kompromiss für schrittweise Einführung und Sonderfälle. |

### 7.2 Kanonische Metafelder

Unabhängig vom Ausgabemodus sollen mindestens folgende technischen Metainformationen vorgesehen werden:

- _transfer_id
- _basket_id
- _topic
- _class
- _oid
- _parent_oid (für Child-/Struktur-Entitäten)
- _path (fachlicher oder technischer Herkunftspfad)
- _seq (Reihenfolge bei wiederholten Elementen)
- _event oder _operation (soweit fachlich relevant, z. B. Delete/Update-Semantik)
- _source_file
## 8. Anforderungen an die Behandlung von Assoziationen

Assoziationen dürfen nicht implizit verloren gehen. Die Abbildung muss so erfolgen, dass Beziehungen downstream eindeutig rekonstruierbar bleiben.

Für einfache Referenzbeziehungen soll eine direkte Ausgabe als Fremdschlüssel-/OID-Feld im Hauptobjekt möglich sein, zum Beispiel in der Form ref_<rolle>_oid.

Für komplexere Beziehungen, mehrwertige Verknüpfungen, n:m-Beziehungen und Assoziationen mit eigenen Attributen soll ein separater Link-Stream vorgesehen werden.

| Variante | Beschreibung | Bewertung |
| --- | --- | --- |
| FK-Shortcut | Referenzen werden als OID-/Ref-Felder direkt in der Haupt-Row geführt. | Einfach und benutzerfreundlich, aber nur für einfache Fälle hinreichend. |
| Link-Stream | Jede Assoziation wird als eigene Zeile mit Rollen, OIDs und Attributen emittiert. | Robusteste Form für komplexe Modelle. |
| Beides | Hauptobjekt enthält einfache Ref-Felder; zusätzlich existiert ein vollständiger Assoziations-Stream. | Empfohlene konfigurierbare Zielrichtung. |

Mindestfelder eines Assoziations-/Link-Streams sollen sein: Assoziationstyp, linke und rechte Objektidentifikatoren, Rollennamen, Basket-/Topic-Bezug sowie gegebenenfalls Assoziationsattribute.

## 9. Anforderungen an die Behandlung von Strukturen

Strukturen und insbesondere LIST/BAG OF STRUCTURE sind die zweite zentrale Schwierigkeit. Anders als in FME sollen sie nicht primär als native Listen im selben Datensatz vorausgesetzt werden, weil Hop keine äquivalente Standardrepräsentation besitzt.

Für einfache, nicht wiederholte Strukturen geringer Tiefe soll Prefix-Flattening möglich sein, zum Beispiel adresse_strasse oder adresse_plz.

Für wiederholte oder tiefere Strukturen soll die Standardstrategie ein Child-Stream sein, der mindestens parent_oid, path, seq und die Strukturattribute enthält.

Zusätzlich soll optional eine serialisierte Form als Fallback möglich sein, etwa JSON-ähnlich oder in einer anderen geeigneten Text-/Binary-Form.

| Strategie | Beschreibung | Empfehlung |
| --- | --- | --- |
| Prefix-Flattening | Einmalige, einfache Strukturen werden in Spalten des Hauptobjekts aufgelöst. | Nur für einfache und stabile Fälle. |
| Child-Stream | Strukturinstanzen werden als eigene Zeilen mit Parent-Bezug ausgegeben. | Bevorzugte Standardlösung für wiederholte oder komplexe Strukturen. |
| Serialized Field | Die Struktur wird als serialisierter Inhalt mitgeführt. | Als Fallback oder ergänzende Ausgabe sinnvoll. |

## 10. Anforderungen an Geometrien, Enumerationen und Vererbung

### 10.1 Geometrien

Geometrien sollen mindestens in einer für Hop einfach verarbeitbaren Standardform ausgegeben werden können. Als Default wird WKT empfohlen, weil es lesbar, testbar und in vielen Zielkomponenten praktikabel ist.

Optional sollen alternative Ausgaben möglich sein, insbesondere WKB/Binary oder weitere zielsystemspezifische Formen.

Bei mehreren Geometrieattributen pro Klasse soll keine stillschweigende Verdrängung stattfinden. Entweder sind mehrere Geometriefelder auszugeben oder es ist ein explizites alternatives Mapping vorzusehen.

### 10.2 Enumerationen

Enumerationen sollen mindestens als stabiler Codewert ausgegeben werden können.

Optional soll zusätzlich oder alternativ eine textuelle Repräsentation verfügbar sein.

### 10.3 Vererbung

Die Behandlung von Vererbung soll nicht implizit bleiben. Das Plugin soll perspektivisch mindestens zwei Strategien unterstützen: eine Superclass-Strategie und eine Subclass-Strategie.

Die konkrete Strategie soll für Nutzer sichtbar und konfigurierbar sein. Die Zugehörigkeit zum konkreten INTERLIS-Typ muss im Output eindeutig erkennbar bleiben.

## 11. Anforderungen an die Plugin-Architektur

Die interne Architektur soll mindestens in die Schichten Modellanalyse, Datenlesen, Mapping und Row-Emission getrennt werden.

Empfohlene technische Dienste sind: ein Model Service zum Kompilieren und Auswerten des INTERLIS-Modells, ein Read Service zum iterativen Lesen der Daten, ein Mapping Service zur Ableitung von Entity-Definitionen und ein Emission Service zur Erzeugung von Hop-Rows.

Diese Trennung ist wesentlich, damit fachliche Mapping-Entscheidungen nicht mit Parser- oder UI-Logik vermischt werden.

Die Hop-seitige Plugin-Struktur soll einem externen Transform-Plugin entsprechen. Die Lösung ist ausdrücklich als externes Plugin zu konzipieren.

## 12. Anforderungen an die Transform-Konfiguration

Die Konfiguration des Transforms soll die wesentlichen Modellierungsentscheidungen explizit machen. Benutzer sollen nicht gezwungen sein, stillschweigende Defaults zu erraten.

Mindestens folgende Konfigurationspunkte sind vorzusehen:

- INTERLIS-Datenquelle
- Modellpfad beziehungsweise Repository-Konfiguration
- optionale Einschränkung auf Topic, Klasse oder Startentität
- Ausgabemodus
- Strategie für Strukturen
- Strategie für Referenzen und Assoziationen
- Strategie für Geometrien
- Strategie für Enumerationen
- Fehlerverhalten (fail, warn, skip oder äquivalent)
- optionale Aktivierung von Meta-Streams und Error-Streams
## 13. Empfohlene Output-Struktur

| Stream/Entität | Zweck | Status |
| --- | --- | --- |
| main | Hauptobjekte/Klassenattribute | Muss |
| structures | Wiederholte oder komplexe Strukturwerte | Muss im relationalen Modus |
| associations | Links, Rollen, Assoziationsattribute | Muss im relationalen Modus |
| meta_baskets | Basket-Metadaten | Soll |
| meta_transfer | Transfer-Metadaten | Soll |
| meta_deleteobjects | Delete-/Operation-Entitäten, soweit relevant | Soll |
| errors | Kontrollierter Fehler- oder Warnkanal | Muss |

Falls die technische Umsetzung mehrere Hop-Ausgänge erschwert, kann alternativ ein unionsartiger Einzelstream mit einem Feld wie _entity_kind vorgesehen werden. Diese Variante ist jedoch für Anwender weniger komfortabel und soll nur zweite Wahl sein.

## 14. Abgrenzungen und bewusst nicht empfohlene Ansätze

- Nicht empfohlen ist ein blindes Voll-Flattening aller Daten in eine einzige sehr breite Row-Struktur.
- Nicht empfohlen ist die rekursive Inline-Expansion sämtlicher Assoziationen in die Hauptobjekte.
- Nicht empfohlen ist eine Ausgabe, bei der komplexe Konstrukte stillschweigend verworfen oder nur implizit im Log erwähnt werden.
- Nicht empfohlen ist eine Schemaerzeugung, die erst während der Verarbeitung opportunistisch entsteht und dadurch instabil oder schwer reproduzierbar wird.
## 15. Inkrementelles Umsetzungsmodell

| Stufe | Inhalt | Ziel |
| --- | --- | --- |
| MVP 1 | Einfache Klassen, Basisattribute, OID, einfache Referenzen als Felder, flache Strukturen bis Tiefe 1, ein Haupt-Output | Früher End-to-End-Nachweis |
| MVP 2 | Modellgetriebenes Schema, mehrere Klassen, Child-Streams für Strukturen, Error-Output | Tragfähige Grundarchitektur |
| MVP 3 | Assoziations-Streams, Geometrieoptionen, Enumerationsoptionen, Meta-Streams, Performance-Optimierung | Produktionsreife für komplexere Modelle |

## 16. Offene Punkte und Unklarheiten

Die genaue Zielmenge der zu unterstützenden INTERLIS-Konstrukte ist noch nicht vollständig spezifiziert. Unklar ist insbesondere, welche Sonderfälle zwingend bereits in der ersten Version abgedeckt werden müssen.

Nicht abschließend geklärt ist, ob und wie viele getrennte Hop-Ausgänge ein einzelner Transform in der geplanten Nutzungsweise praktisch bereitstellen soll oder ob dafür ergänzende Musterkomponenten nötig sind.

Die gewünschte Zielrepräsentation für Geometrien ist noch nicht fachlich fixiert. WKT ist eine pragmatische Empfehlung, aber nicht zwingend die einzige sinnvolle Zielwahl.

Nicht festgelegt ist, welche INTERLIS-Dateiformate und welche Interoperabilitätsvarianten zwingend im ersten Ausbauschritt unterstützt werden müssen.

Offen ist ferner, ob Delete-/Update-Semantik, Validierungsinformationen und technische Ereignisse bereits in der ersten Fassung als vollwertige Meta-Streams modelliert werden sollen.

Die Anforderungen an die Benutzeroberfläche des Hop-Transforms, etwa Vorschau, Schema-Preview oder modellbasierte Assistenz, sind noch nicht genauer beschrieben.

Für die Vererbungsstrategie ist noch nicht entschieden, welcher Modus Default sein soll und ob beide Strategien bereits zu Beginn zwingend implementiert werden müssen.

## 17. Zusammenfassende Architekturentscheidung

Die derzeit belastbarste Leitentscheidung lautet wie folgt: INTERLIS-Daten sollen in Apache Hop primär über ein modellgetriebenes, relational-hybrides Zwischenmodell verarbeitet werden. Einfache Attribute werden in Haupt-Streams ausgegeben, Referenzen und Assoziationen als OID-Felder und/oder Link-Streams, wiederholte oder komplexe Strukturen als Child-Streams, schwierige Sonderfälle optional zusätzlich serialisiert. Die Prinzipien aus ili2fme sind fachlich wertvoll, die FME-spezifische Repräsentation mit Listenattributen und nativer Feature-Geometrie ist für Hop jedoch nicht 1:1 übertragbar.

## 18. Verwendbarkeit dieses Dokuments

Dieses Dokument ist ein fachlich-technischer Anforderungsentwurf. Es ist geeignet als Grundlage für eine spätere Feinspezifikation, ein Architekturkonzept, ein Implementierungs-Backlog oder eine strukturierte Projektentscheidung. Aussagen, die derzeit noch offen sind, wurden bewusst als offene Punkte ausgewiesen und nicht künstlich vorentschieden.
