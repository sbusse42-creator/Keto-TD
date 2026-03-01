# Keto Drop Defense (PWA)

Kleines Offline-PWA-Spiel (statische Webseite) für Android/Chrome:
- Lebensmittel regnen herunter
- Antippen: vernichten (0 Punkte)
- Durchkommen lassen: +/- Punkte je Keto-Tauglichkeit
- Sieg: +100 / Niederlage: -100

## Lokal starten (Service Worker benötigt http/https)
```bash
python -m http.server 8080
```

Dann öffnen:
- Desktop: http://localhost:8080
- Android im selben WLAN: http://<PC-IP>:8080

## PWA installieren (Android)
In Chrome: Menü → **Zum Startbildschirm hinzufügen** / **Installieren**.

## Deployment (Azure Static Web Apps)
Repo in GitHub pushen und im Azure Portal eine Static Web App mit GitHub verbinden.
Die Workflow-Datei liegt unter `.github/workflows/azure-static-web-apps.yml`.
