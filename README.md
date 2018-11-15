# Login API


Es gibt 2 Authentifizierungs-Möglichkeiten um auf das EUROPACE System zuzugreifen:
* [Authentiﬁzierung mit Nutzername und Passwort](#nutzername-und-passwort)
* Anmeldung via _Silent Sign On (SSO)_

Bei Fragen und Anregungen entweder ein Issue in GitHub anlegen oder an [devsupport@europace2.de](mailto:devsupport@europace2.de) schreiben.

# Nutzername und Passwort

Formularbasiertes Silent-Sign-On
-----------------------------------------

Neben der Standard-Login-Seite, stellt EUROPACE 2 seinen Partner zwei weitere Wege zur Authentifizierung bereit.

### Partner Html-Login-Box

Unter https://europace2.de/partnermanagement/partner-login-box.html steht eine minimale
HTML Seite zur Verfügung, welche die Eingabefelder für Benutzer und Passwort sowie
eine Login-Button enthält. Die Seite enthält zusätzliche Komfort-Funktionen für Benutzer:

* Setzen der Tastatureingabe auf das erste, nicht-ausgefüllte Feld
* Speichern des Benutzernamen und anzeigen des selben nach Wiederaufruf der Seite

Diese Seite kann mittels iFrame direkt eingebettet werden:

```
<iframe src="https://europace2.de/partnermanagement/partner-login-box.html"
width="366px" height="250px" style="margin: 0px 0px 0px 0px;"></iframe>

```

Alternativ dient diese Seite als ''Kopiervorlage'' zum direkten Einbetten in die
Partner-Webseite. Folgende Elemente sind zu Kopieren:

* CSS Stylesheet (Zeile 4-62)
* Formular inkl. Javascript-Komfortfunktionen (Zeile 67-141)

Die Html Seite hat keinerlei externe Abhängigkeiten auf CSS oder Javascript Bibliotheken.

![Eingebettete Login-Box von EUROPACE2](images/partner_login_box.png?raw=true)


### Einloggen per HTTP POST Request

Eine durch den Partners bereitgestelltes Login-Formular muss zur Authentiﬁzierung an
Europace 2 folgenden Request absenden.

Request

````
POST https://www.europace2.de/partnermanagement/login.do?oeffne=%2FvorgangsManagementFrontend
---
Header:
Content-type: application/x-www-form-urlencoded;charset=UTF-8
---
Body (Form Data URL encoded):
username=Mustermann@beispiel.de&password=4711

````

Der URL Parameter ''oeffne'' ist optional. Dieser kann benutzt werden,
um die Zieladresse nach erfolgreichem Login festzulegen.

Response (korrekte Anmeldedaten):

````
HTTP/1.1 200 OK
Content-Type: text/plain
Set-Cookie: sessionId=3f8a86fa02128b5265de9f1ec4b3c4be; Path=/
````

Response (fehlerhafte Anmeldedaten):

````
HTTP/1.1 401 Unauthorized
````


Eine manuelle Überprüfung ist z.B. mittels des Kommandozeilen-Programmes "curl" möglich:

````
curl -v --insecure --header "Content-type: application/x-www-form-urlencoded;charset=UTF-8" --data "username=Mustermann@beispiel.de&password=4711" https://www.europace2.de/partnermanagement/login.do
````

### Ausloggen

Der Logout kann ebenenfalls über einen HTTP POST Request erfolgen.
Dabei muss der nach der Authentifizierung gelieferte Cookie  
"sessionId" übertragen werden.  

Request

````
POST https://www.europace2.de/partnermanagement/logout

Header:
Cookie: sessionId=EineLangeSessionId
````

Response

````
HTTP/1.1 302 Found
````

Manuelle Überprüfung mittels "curl":

````
curl -v --insecure -X POST --cookie "sessionId=3f8a86fa02128b5265de9f1ec4b3c4be" https://www.europace2.de/partnermanagement/logout
````

# Silent Sign On


_Silent Sign On_ in ein Verfahren, das es ermöglicht, einen Benutzer von einer Webseite zu BaufiSmart weiter zu leiten, ohne das sich dieser erneut anmelden muss.


EUROPACE 2 nutzt dafür den Internet Standard [JWT](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32)
und [JWS](https://tools.ietf.org/html/draft-ietf-jose-json-web-signature-41)
mit [RSA](http://de.wikipedia.org/wiki/RSA-Kryptosystem) als Signaturverfahren.

Dieses Dokument erläutert Ihnen die verwendeten technischen Standards und welche Schritte zu einer erfolgreichen Anbindung notwendig sind.

### Übersicht der notwendigen Schritte als Shell Skripte


Folgende Script Aufrufe spielen den Prozess exemplarisch durch. Benötigt werden der API-Key und die Partner-Id der Organisation (issuer) sowie die Partner-Id des einzuloggenden Benutzers.

Es muss Java 1.8 installiert sein.

```
./generate-key.sh
./upload-key.sh  <api-key> <partnerId_des_key_issuers>
./login-via-sso.sh <partnerId_des_einzuloggenden> <partnerId_des_key_issuers> [redirectTarget]
```

Die Skripte [generate-key.sh](./generate-key.sh) [upload-key.sh](./upload-key.sh) [login-via-sso.sh](./login-via-sso.sh)
automatisieren die manuellen Schritte, um mit OpenSSL und curl die Keys zu erzeugen und hochzuladen.


Erzeugung der RSA Keys
----------------------

```
openssl genrsa 2048 > rsa.private
openssl pkcs8 -topk8 -inform PEM -outform PEM -in rsa.private  -nocrypt > private-key.pem
openssl rsa -inform PEM -in rsa.private -pubout > public-key.pem
rm rsa.private
```

Hinterlegen des Public RSA Key in EUROPACE 2
--------------------------------------------

Der öffentliche RSA key muss in EUROPACE 2 hinterlegt sein, um die Signatur des JWT zu überprüfen. Der öffentliche Schlüssel kann per HTTP hochgeladen werden:

```
curl -X PUT
	 -H "Accept: application/json"
	 -H "X-ApiKey: ${apiKey}"
	 -H "X-PartnerId: ${partnerId}"
	 -H "X-TraceId: `date "+%Y-%m-%dT%H:%M:%S"`"
	 -H "Content-Type: text/plain;charset=utf-8"
	 --data @public-key.pem
	 https://www.europace2.de/partnermanagement/partner/${issuerId}/sso-pub-key
```

apiKey und partnerId müssen einer Organisationseinheit identifizieren, die "Einstellungsrechte" auf den _issuerId_ hat.
Die _issuerId_ entspricht der partnerId, für die der Key hinterlegt werden soll.

Erzeugen eines Json Web Tokens (JWT)
-------------------------------------

### Token-Attribute und deren Bedeutung

Für ein Anmelde-Token werden nur wenige Attribute benötigt.
Diese werden sowohl im Token-Header als auch im Payload eingesetzt.
Die Namen der Attribute sind an die Spezifikation von
[JWT](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32)
und [JWS](https://tools.ietf.org/html/draft-ietf-jose-json-web-signature-41) angelehnt.

#### Token Header

Die JSON Struktur des Token Headers sieht wie folgt aus:

````json
{
  "iss": "issuerId",
  "alg": "RS256"
}
````

Das Feld _iss_ beschreibt die PartnerId der unterschreibenden Person oder Organisation (Bürge).
Das Feld _alg_ beschreibt, dass der _RS256_ Algorithmus zum Signieren verwendet wird. Es wird ausschließlich nur dieser Algorithmus unterstützt.


#### Token Payload

````json
{
  "sub": "partnerToLogonId",
  "exp": 1424190490
}
````

Das Feld _sub_ beschreibt die PartnerId der einzuloggenden Person oder Organisation.
Das Feld _exp_ beschreibt den Ablaufzeitpunkt bzw. maximale Gültigkeit dieses Tokens in Form der [Unixzeit](http://de.wikipedia.org/wiki/Unixzeit)

#### Absicherung durch die Hierarchie

Es ist nur erlaubt, Partner, die in der Europace Struktur untergeordnet sind, sowie sich selbst, als vertrauenswürdig einzustufen. Das heißt, dass der
Bürge (Issuer) im Baum immer über der einzuloggenden Person (Subject) stehen muss.
Es ist nicht erlaubt, einer Person oder Organisation, in einem Nachbarbaum den Login auf die Plattform zu ermöglichen.

### Java Beispiel

Für manuelle, explorative Tests der SSO Schnittstelle sowie als "Kopiervorlage" für Java basierte clients steht ein Java basiertes Testprojekt bereit.

Folgende Komponenten müssen installiert sein:

- [Java8](http://www.java.com/en/download/index.jsp)
- [Maven3](http://maven.apache.org/download.cgi)


1) Github Projekt auschecken:

```
git clone https://github.com/hypoport/jwt-toolbox.git
```

2) Projekt kompilieren und all-inclusive JAR erstellen.

```
cd jwt-toolbox
mvn package
```

3) Token erzeugen

```
java -jar target/jwt-toolbox-jar-with-dependencies.jar private-key.pem ${partnerToLogonId} ${issuerId}
```


### Online Tool zur Visualisierung

Die erzeugten Token sind [Base64url](https://tools.ietf.org/html/rfc4648#page-7) kodiert.
Zur einfachen visuellen Überprüfung kann man das Online-Tool auf [jwt.io](http://jwt.io) verwenden.



Einloggen via JWT mit Redirect
------------------------------

Der soeben erzeugte Token kann nun zur Anmeldung an EUROPACE 2 verwendet werden.

Nun kann im Browser folgende URL aufgerufen werden:

```
https://www.europace2.de/partnermanagement/login?redirectTo=/uebersicht&authentication=${jwt}
```

Aufruf mit _curl_:

```
curl -v -X POST "https://www.europace2.de/partnermanagement/login?redirectTo=/uebersicht&authentication=`java -jar jwt-toolbox.jar private-key.pem ${partnerToLogonId} ${issuerId}`"
```

Alternativ kann auch ein POST request mit gesetztem X-Authentication header verwenden werden:

```
POST /partnermanagement/login?redirectTo=/uebersicht&authentication=${jwt} HTTP/1.1
Host: www.europace2.de
X-Authentication: ${jwt}
X-TraceId: Eine_TraceId_zB_Datum_rueckwaerts
```

Aufruf mit _curl_:

```
curl -v -X POST -H "X-Authentication: `java -jar jwt-toolbox.jar private-key.pem ${partnerToLogonId} ${issuerId}`" "https://www.europace2.de/partnermanagement/login?redirectTo=/uebersicht"
```


Response:

```
HTTP/1.1 302 FOUND
Location: /uebersicht
Set-Cookie: sessionId=dd800897698f8e2637d5c39e33083764
```


### Schematischer Ablauf

Das folgende Sequenzdiagramm erläutert das Zusammenspiel der Kollaborateure in der HTTP Netzwerkebene:

![Sequenzdiagramm für die EUROPACE 2 Anmeldung via Silent Sign On](images/Sequence.png?raw=true)
