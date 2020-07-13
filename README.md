# Login API ("legacy")

⚠️ Hinweis: die Autorisierung für die Europace APIs und SSO an der Plattform werden aktuell grundlegend überarbeitet. Es gibt einen neuen OAuth-fähigen [Autorisierungs-Server](https://github.com/europace/authorization-api) dessen Verwendung wir empfehlen. Bei Fragen bitte an devsupport@europace2.de schreiben.


Es gibt 2 Authentifizierungs-Möglichkeiten um auf das EUROPACE System zuzugreifen:
* [Authentiﬁzierung mit Nutzername und Passwort](#nutzername-und-passwort)
* [Anmeldung via _Silent Sign On (SSO)_](#silent-sign-on)

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

![Eingebettete Login-Box von EUROPACE2](https://raw.githubusercontent.com/europace/login-api/master/images/partner_login_box.png)


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


_Silent Sign On_ in ein Verfahren, dass es ermöglicht, einen Benutzer von einer Webseite zu BaufiSmart weiter zu leiten, ohne das sich dieser erneut anmelden muss.


EUROPACE 2 nutzt dafür den Internet Standard [JWT](https://tools.ietf.org/html/draft-ietf-oauth-json-web-token-32).

Im wesentlichen werden dafür 2 Schritte durchgeführt:
1. Ein JWT (Access Token) an der Login API anfordern
2. Aufruf einer BaufiSmart URL mit dem Access Token als Request-Parameter.


## JWT anfordern

Die Authentifizierung läuft über den OAuth2 Flow vom Typ ressource owner password credentials flow. https://tools.ietf.org/html/rfc6749#section-1.3.3

Bei den Credentials kann entweder
* der API Key und die Partner ID verwendet werden *oder*
* Benutzernamen (E-Mail-Adresse) und Passwort eines Nutzers, der Zugzung zu BaufiSmart oder KreditSmart hat.

### Schritte
1. Absenden eines POST Requests auf den Login-Endpunkt mit Username und Password. https://api.europace.de/login
2. Aus der JSON-Antwort das JWT (access_token) entnehmen
3. Bei weiteren Requests muss dieses JWT als Authorization mitgeschickt werden.

### Sonderfall: einloggen mit einer untergeordneten Partner ID
Um zu verhindern, dass für jeden Nutzer ein API Key angefordert werden muss, oder dass Passwörter gespeichert werden, gibt es auch die Möglichkeit, SSO für eine beliebige
Unterplakette durchzuführen. Dafür muss ein gesonderter Login-Endpunkt verwendet werden.

Man benötigt dafür

* den API Key der obersten Plakette in der Struktur (`SECRET` im Beispiel unten)
* die Partner ID der Plakette die sich einloggen soll (`AB123` im Beispiel unten)

````
POST https://www.europace2.de/partnermanagement/login.api
---
Header:
x-partnerid: AB123
x-apikey: SECRET
---
````
Im Respnse wird ein Cookie gesetzt namens `authentication`. Dieses ist ein JWT und kann im nächsten Schritt verwendet werden.

Übrigens befindet sich ein Beispiel hierfür auch in unserer [Postman Collection](https://github.com/europace/api-schnellstart)

## Einloggen mit Access Token als Request Parameter


Der soeben erzeugte JWT kann nun zur Anmeldung an EUROPACE 2 verwendet werden.

Nun kann im Browser folgende URL aufgerufen werden:

```
https://www.europace2.de/partnermanagement/login?redirectTo=/uebersicht&authentication=${jwt}
```

Alternativ kann auch ein POST request mit gesetztem X-Authentication header verwenden werden:

```
POST /partnermanagement/login?redirectTo=/uebersicht&authentication=${jwt} HTTP/1.1
Host: www.europace2.de
X-Authentication: ${jwt}
X-TraceId: Eine_TraceId_zB_Datum_rueckwaerts
```



Response:

```
HTTP/1.1 302 FOUND
Location: /uebersicht
Set-Cookie: sessionId=dd800897698f8e2637d5c39e33083764
```


## Nutzungsbedingungen
Die APIs werden unter folgenden [Nutzungsbedingungen](https://developer.europace.de/terms/) zur Verfügung gestellt.
