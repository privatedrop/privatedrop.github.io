---
title: Breaking and Fixing Apple AirDrop
---

Diese Website ist auch [**auf Englisch**]({% link index.md %}) verfügbar.

## AirDrop Einmaleins

Apple AirDrop ist ein Dienst zum Teilen von Dateien, der es Nutzer\*innen ermöglicht, Photos und andere Medien über eine direkte Wi-Fi-Verbindung von einem Apple-Gerät zum anderen zu übertragen. Da Nutzer\*innen sensible Daten üblicherweise nur mit Personen teilen, die sie bereits kennen, zeigt AirDrop standardmäßig nur Empfänger aus dem Adressbuch an. Um zu bestimmen, ob die jeweils andere Partei ein bekannter Kontakt ist, nutzt AirDrop einen gegenseitigen Authentifizierungsmechanismus, der die Telefonnummer und Email-Adresse eines Nutzers bzw. einer Nutzerin mit Einträgen des Adressbuchs der anderen Partei abgleicht.

## Das Problem: Preisgabe von Telefonnummern und E-Mail-Adressen

Wir haben _zwei_ schwerwiegende Datenschutzlücken in diesem Authentifizierungsmechanismus identifiziert. Konkret konnten wir zeigen, dass Angreifer an die Telefonnummern und E-Mail-Adressen von Apple-Nutzer\*innen gelangen können -- selbst ohne jegliches Vorwissen über ihre Opfer. Dafür benötigen Angreifer lediglich ein Wi-Fi-fähiges Gerät und die physische Nähe zu Personen mit Apple-Geräten.

Die identifizierten Probleme sind auf die Verwendung von sogenannten Hash-Funktionen zurückzuführen, die Apple nutzt, um Kontaktdaten (Telefonnummern und E-Mail-Adressen) während der Authentifizierung zu "verschleiern". Allerdings ist bereits hinlänglich bekannt, dass [Hashing keinen Privatsphäre-Schutz bei Kontaktermittlung bietet](https://contact-discovery.github.io/de), da die Hash-Werte von Telefonnummern sehr schnell mittels einfach Techniken wie beispielsweise Brute-Force-Angriffen zurückgerechnet werden können.

### Schwachstelle #1: Sender geben Kontaktdaten preis

Während der AirDrop Authentifizierung gibt der Sender immer seine eigenen (gehashten) Kontaktdaten als Teil der initialen _Discover_-Nachricht preis. Ein bösartiger Empfänger kann daher an alle (gehashten) Kontaktdaten des Senders gelangen, ohne dass irgendwelche Kenntnisse über das Opfer vorhanden sein müssen. Um an diese Kontaktdaten zu gelangen, muss ein Angreifer einfach (z.B. an einem belebten Ort) warten, bis ein mögliches Opfer nach AirDrop-Empfängern sucht, also das "Teilen"-Menü öffnet.

Nachdem Angreifer an (gehashte) Kontaktdaten gelangt sind, können die Telefonnummern und E-Mail-Adressen zu einem beliebigen späteren Zeitpunkt zurückgerechnet werden. Wie [vorausgehende Veröffentlichungen](https://encrypto.de/papers/HWSDS21.pdf) gezeigt haben, ist es möglich Telefonnummern innerhalb von Millisekunden zurückzurechnen. Für E-Mail-Adressen ist das weniger trivial, aber dennoch über sogenannte Wörterbuch-Angriffe möglich, bei denen häufige E-Mail-Formate wie vorname.nachname@{gmail.com,yahoo.com,...} geprüft werden. Alternativ können Angreifer die Informationen aus [massiven Datenlecks](https://www.businessinsider.com/stolen-data-of-533-million-facebook-users-leaked-online-2021-4) oder spezielle [Onlinedienste zum Nachschlagen von gehashten E-Mail-Adressen](https://web.archive.org/web/20191211152224/https://datafinder.com/products/email-recovery) nutzen.

Dieser Angriff wurde ebenfalls und unabhängig vom [Apple Bleee](https://hexway.io/research/apple-bleee/) Projekt entdeckt und im Juli 2019 veröffentlicht, kurz nachdem wir Apple darüber im Mai 2019 informiert hatten.

### Schwachstelle #2: Empfänger geben Kontaktdaten preis

AirDrop Empfänger geben ihre (gehashten) Kontaktdaten als Antwort auf die initiale Nachricht des Senders preis, falls sie _irgendeine_ der Telefonnummern oder E-Mail-Adressen des Senders kennen. Ein bösartiger Sender kann daher ohne jegliches Vorwissen über den Empfänger an _alle_ Kontaktdaten des Empfängers kommen (inklusive Telefonnummer) -- falls der Empfänger den Sender kennt.

Dabei ist wichtig zu beachten, dass ein bösartiger Sender den Empfänger nicht mal kennen muss: eine populäre Person innerhalb eines bestimmten Kontextes (beispielsweise Geschäftsführer einer Firma) kann dieses Problem ausnutzen, um an alle (privaten) Kontaktdaten anderer Menschen zu gelangen, die die populäre Person in ihrem Adressbuch gespeichert haben (beispielsweise Angestellte der Firma).

### Proof-of-Concept Angriffe

Wir demonstrieren die Angriffe zum Ausnutzen der beiden Schwachstellen mit einer Proof-of-Concept-Implementierung die öffentlich auf [GitHub](https://github.com/seemoo-lab/opendrop/blob/poc-phonenumber-leak/README.PoC.md) verfügbar ist. Die Implementierung kombiniert [OpenDrop](https://github.com/seemoo-lab/opendrop), einer Open-Source Implementierung von AirDrop, mit [RainbowPhones](https://github.com/contact-discovery/rt_phone_numbers), einem Open-Source Cracking-Tool das auf das Finden von Telefonnummern optimiert ist.

## Unsere Lösung: PrivateDrop

Wir haben eine praktikable Lösung entwickelt, die das unsichere AirDrop ersetzen könnte. PrivateDrop basiert auf kryptographischen Protokollen für "Private Set Intersection", also zur sicheren Berechnung einer Schnittmenge aus vertraulichen Datensätzen. Mit dieser Methode kann die gegenseitige Authentifizierung durchgeführt werden, ohne angreifbare Hash-Werte austauschen zu müssen. Unser Prototyp für iOS und macOS zeigt, dass unser Ansatz zur Privatsphäre-schützenden gegenseitigen Authentifizierung effizient genug ist, um AirDrops vorbildliche Nutzererfahrung beizubehalten, da die benötigte Zeit für die Authentifizierung weit unter einer Sekunde liegt.

Die Implementierung von PrivateDrop ist öffentlich auf [GitHub](https://github.com/seemoo-lab/privatedrop) verfügbar.

## Responsible Disclosure

Wir haben Apple über die Datenschutzlücken im Mai 2019 mittels "Responsible Disclosure" informiert und unsere Lösung PrivateDrop im Oktober 2020 geteilt. Stand 20. April 2021 hat Apple nicht erkennen lassen, dass seitens der Firma an einer Lösung gearbeitet wird.

Daher sind **Apple-Nutzer\*innen derzeit weiterhin anfällig für die genannten Angriffe.** Die einzige Möglichkeit sich zu schützen besteht derzeit darin, die AirDrop-Erkennung in den Systemeinstellungen zu deaktivieren und davon abzusehen, das "Teilen"-Menü zu öffnen.

## Publikation

- [HHSSW21] **_PrivateDrop: Practical Privacy-Preserving Authentication for Apple AirDrop_** von [Alexander Heinrich](https://www.informatik.tu-darmstadt.de/seemoo/team_seemoo/alexander_heinrich/alexander_heinrich.en.jsp), [Matthias Hollick](https://www.informatik.tu-darmstadt.de/seemoo/team_seemoo/matthias_hollick/index.en.jsp), [Thomas Schneider](https://encrypto.de/schneider), [Milan Stute](https://www.informatik.tu-darmstadt.de/seemoo/team_seemoo/milan_stute/milan_stute.en.jsp) und [Christian Weinert](https://encrypto.de/weinert) in [30th USENIX Security Symposium (USENIX Security'21)](https://www.usenix.org/conference/usenixsecurity21). Paper verfügbar als **[pre-print](https://www.usenix.org/system/files/sec21fall-heinrich.pdf)**. Implementierung verfügbar auf **[GitHub](https://github.com/seemoo-lab/privatedrop)**. Proof-of-Concept Angriffe verfügbar auf **[GitHub](https://github.com/seemoo-lab/opendrop/blob/poc-phonenumber-leak/README.PoC.md)**.
