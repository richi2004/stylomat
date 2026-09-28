# CLAUDE.md – Stylomat

## Worum es geht

**Stylomat** ist eine persönliche Kleiderschrank-App, die **richtig gute Outfits** vorschlägt, nicht nur "passt schon".
Fertige Apps (Acloset, Whering, Fits usw.) scheitern an Passform, Material, Proportionen und mutigen Kombinationen,
weil sie das aus Fotos raten. Stylomat **erfasst diese Eigenschaften explizit** und lässt ein
multimodales Large Language Model (LLM) auf Basis von Fotos + strukturierten Daten + Stilprofil stylen.

**Erster und vorerst einziger Nutzer: Richi.** Genutzt wird die App auf dem iPhone. Die Architektur ist
mehrnutzerfähig vorbereitet (siehe Architekturregeln), es gibt aber genau einen Account und keine Registrierung.

## Kernprinzipien für gute Outfits (gehören in die Prompts, nicht in Code-Regeln)

1. **Ein Fokuspunkt pro Outfit.** Maximal ein Statement-Teil, der Rest ordnet sich unter.
2. **Silhouette vor Farbe.** Passform und Proportionen (Volumen oben vs. unten, Hosenweite zu Schuhvolumen) entscheiden zuerst.
3. **Texturkontrast** (z. B. Strick zu Leder, grober Denim zu glatter Baumwolle) wertet gedeckte Outfits auf.
4. **Farben nah am Gesicht** passend zum Farbtyp des Nutzers.
5. **Styling-Moves zählen wie Accessoires**: Ärmel krempeln, Hose umschlagen, halb einstecken, offen tragen, sichtbares Layering.
6. **Anlass mit Kontext** (Ort + Leute + Vibe), nicht nur ein Schlagwort.
7. **Frisur und Bart gehören zur Silhouette.** Bart gibt Gesicht/Hals Gewicht (Kragenwahl: offen vs. Rollkragen),
   Haarvolumen beeinflusst die Proportionen, Bart- und Haarfarbe zählen bei Farben nah am Gesicht, und der
   Grooming-Stil muss zum Vibe passen (oder bewusst kontrastieren).

## Tech-Stack (entschieden)

- **Backend:** Python + FastAPI
- **Datenbank:** SQLite als Single Source of Truth (SSOT), Zugriff nur über eine Repository-Schicht
- **Frontend:** mobil-optimiertes Web-Frontend als **Progressive Web App (PWA)**, auf dem iPhone über
  Safari → Teilen → "Zum Home-Bildschirm" installiert. Braucht: `manifest.json` (`display: standalone`, Name "Stylomat"),
  App-Icons inkl. `apple-touch-icon`, Meta-Tag `apple-mobile-web-app-capable`, Layout für iPhone-Breite inkl. Safe Areas (Notch).
  Die Web-App ist **nur ein Client** der JSON-API (siehe Architekturregel "API-first"). Später kommt eine native
  iOS-App (SwiftUI, Mac ist vorhanden) als zweiter Client auf dieselbe API dazu – Backend und Daten bleiben gleich.
  Frontend-Technik: schlichtes HTML/CSS/JavaScript ohne schweres Framework, als statische Dateien von FastAPI ausgeliefert.
- **Bildverarbeitung:** Pillow + `pillow-heif` (iPhone-Fotos kommen ggf. als HEIC)
- **LLM:** Anthropic API (Modell per Konfiguration, z. B. `claude-sonnet-5`), Vision-fähig
- **Hosting:** bestehender Hetzner-Server (siehe Abschnitt Hosting)

## Hosting (Hetzner)

- Läuft auf demselben Hetzner-Server wie StudyOS, aber **komplett getrennt**: eigenes Verzeichnis, eigener
  systemd-Service, eigene SQLite-Datei, eigene `.env`, eigener Port. Kein gemeinsamer Code, keine gemeinsame Datenbank.
- Server: Hetzner CX23 (2 vCPU, 4 GB RAM, Ubuntu), IPv4 91.99.66.137.
- **Reverse Proxy: Caddy** (läuft bereits für StudyOS). Stylomat bekommt einen eigenen Block in `/etc/caddy/Caddyfile`,
  der auf den lokalen Port von Stylomat zeigt, z. B.:
  `STYLOMAT-HOSTNAME { reverse_proxy 127.0.0.1:8010 }`
  Danach `sudo systemctl reload caddy`. Den bestehenden StudyOS-Block NICHT anfassen.
- Die FastAPI-App lauscht nur auf `127.0.0.1` (nicht öffentlich), erreichbar ausschließlich über Caddy, **nur HTTPS**
  (Caddy holt das Zertifikat automatisch).
- Hostname: noch offen. Braucht einen DNS-Eintrag (A-Record auf 91.99.66.137), z. B. eine Subdomain der StudyOS-Domain.
  Übergangslösung ohne eigene Domain: `stylomat.91-99-66-137.sslip.io` (zeigt automatisch auf die IP, Caddy bekommt dafür ein Zertifikat).
- Bilder liegen außerhalb des Web-Roots und werden **nie** als öffentliche statische Dateien ausgeliefert
  (siehe Login).
- Backup: SQLite-Datei + Bildordner regelmäßig sichern (z. B. täglicher Cronjob in ein Backup-Verzeichnis).

## Login & Sicherheit (klein, aber sauber)

- **Genau ein Account**, angelegt per Kommandozeilen-Befehl auf dem Server (z. B. `python -m stylomat.create_user`).
  Keine Registrierungsseite, kein "Passwort vergessen".
- Passwort nur als Hash speichern (**argon2**).
- Session-Cookie: `HttpOnly`, `Secure`, `SameSite=Strict`, Laufzeit 30 Tage (damit man sich auf dem iPhone nicht ständig neu einloggen muss).
- Die Authentifizierung sitzt in einer eigenen Abhängigkeit (`auth.py`, FastAPI-Dependency), damit später für die native
  App zusätzlich ein Token-Login (`Authorization: Bearer ...`) ergänzt werden kann, ohne die Endpoints anzufassen.
  In Stufe 1 wird nur das Session-Cookie gebaut.
- **Rate-Limit beim Login:** max. 5 Fehlversuche pro 15 Minuten pro IP, danach Sperre.
- **Alle Routen außer `/login` sind geschützt – auch die Bilder.** Bilder werden über einen Endpoint ausgeliefert, der die Session prüft.
- **EXIF-Daten beim Upload entfernen** (iPhone-Fotos enthalten sonst u. a. den GPS-Standort).
- API-Schlüssel nur in der `.env` auf dem Server, nie im Frontend. Eigener Schlüssel nur für Stylomat,
  **monatliches Ausgabenlimit in der Anthropic-Console setzen**.

## Architekturregeln (nicht verhandelbar)

- **API-first:** Das Backend liefert ausschließlich JSON unter `/api/v1/...` (plus Bild-Endpoints). Keine
  serverseitig gerenderten Seiten, keine Logik im Frontend, die ein zweiter Client nachbauen müsste
  (z. B. Outfit-Validierung, Prompt-Aufbau, Bildverarbeitung passieren im Backend). Die Web-App nutzt nur
  Endpoints, die auch die spätere native App nutzen kann. Bild-Uploads als `multipart/form-data`.
  Die automatische API-Doku von FastAPI (`/docs`) ist nur nach Login erreichbar.
- **Jede Tabelle hat eine `user_id`**, auch wenn es vorerst nur einen Nutzer gibt.
- **Repository-Schicht:** Kein SQL außerhalb von `repository.py`.
- **Bild-Speicher-Schicht:** Bilder nur über `image_store.py` speichern/laden (heute lokal, später Cloud austauschbar). Keine Dateipfade quer durch den Code.
- **LLM-Fassade:** Alle LLM-Aufrufe laufen über `llm.py`. Dort wird jeder Aufruf mit `user_id`, Zweck und Token-Verbrauch geloggt.
- **Prompts liegen als eigene Dateien** unter `prompts/` und nicht als Strings im Code, damit sie leicht verbessert werden können.
- Migrationen nur additiv (keine Spalten löschen/umbenennen).

## Bilder: Wege ins System

Ein Kleidungsstück kommt auf zwei Wegen rein:

1. **Foto vom iPhone** (Kamera oder Fotomediathek). Empfohlener Weg für freigestellte Bilder: in der Fotos-App lange
   auf das Kleidungsstück drücken → iOS stellt es frei → "Teilen" → in Stylomat hochladen. Uploads ohne Freistellung
   sind ebenfalls erlaubt.
2. **Import aus dem Internet:** Nutzer fügt entweder eine direkte Bild-URL oder die URL einer Produktseite ein.
   Bei einer Produktseite wird nur das Hauptbild übernommen (`og:image` im HTML). Kein weiteres Auslesen der Seite.
   Sicherheitsregeln für den Import: nur `http`/`https`, keine Adressen im privaten Netz/localhost,
   Content-Type muss ein Bild sein, max. 15 MB, Timeout 10 Sekunden.

Für alle Bilder: HEIC → JPEG umwandeln, EXIF entfernen, gespeichert mit max. 1600 px langer Seite,
an das LLM gehen max. 1024 px (spart Kosten).

Automatische Hintergrund-Entfernung auf dem Server (`rembg`) wird in Stufe 1 **nicht** gebaut – erst, wenn sich
im Alltag zeigt, dass sie fehlt.

## Datenmodell

**user** – id, name, password_hash, erstellt_am

**login_attempt** – id, ip, erfolgreich (ja/nein), zeitpunkt

**style_profile** – user_id, groesse_cm, statur (Freitext, z. B. "breite Schultern, athletischer Oberkörper"),
proportionen (z. B. "eher lange Beine"), hautton (warm/kühl/neutral), haarfarbe, augenfarbe, gesichtsform,
vibe_mix (Freitext, z. B. "70 % clean, 30 % rugged"), lieblingsfarben, no_gos

**grooming_state** (Frisur & Bart, mit Historie – der neueste Eintrag gilt) – id, user_id,
portrait_bild (frontal, Tageslicht), frisur_laenge, frisur_volumen, frisur_stil,
bart_laenge (keiner/stoppel/kurz/mittel/voll), bart_form, bart_dichte, bart_farbe, erfasst_am

**inspiration_image** – id, user_id, bild, notiz ("mag daran: Farbkombi")

**item** (Kleidungsstück) – id, user_id, bild, bild_quelle (foto/url), quelle_url (nur bei Import),
kategorie (oberteil/unterteil/schuhe/jacke/layer/accessoire), unterkategorie (z. B. "Overshirt"),
farben, muster, material, textur (glatt/grob/glänzend/matt), stoffgewicht (leicht/mittel/schwer),
passform (slim/regular/relaxed/oversized), laenge_schnitt (z. B. cropped, straight, tapered, wide),
schuhvolumen (schlank/mittel/klobig – nur Schuhe), waschung_zustand (z. B. "raw denim", "vintage washed"),
rolle (statement/basic), formalitaet (1–5), waerme (1–5),
tragevarianten (Liste: offen, eingesteckt, halb eingesteckt, gekrempelt, als Layer darunter, als Layer darüber),
kaufdatum, notiz, aktiv (ja/nein)

**outfit** – id, user_id, name, anlass_ort, anlass_leute, anlass_vibe, wetter, begruendung (vom LLM),
styling_moves (vom LLM), score (1–10, nur bei bewerteten Outfits), quelle (generiert/swipe/hero_piece/kandidat), gespeichert (ja/nein), erstellt_am

**outfit_item** – outfit_id, item_id

**feedback** – id, user_id, outfit_id, bewertung (hoch/runter), kommentar (z. B. "Hose zu weit zur Jacke"), erstellt_am

**wear_log** – id, user_id, outfit_id, getragen_am, anlass

**mirror_check** – id, user_id, outfit_id (optional), selfie_bild, llm_feedback, erstellt_am

**llm_call_log** – id, user_id, zweck, modell, input_tokens, output_tokens, erstellt_am

## Stufen

### Stufe 1 – Wochenende (MVP)
0. **Grundgerüst:** Login, PWA-Installation aufs iPhone, Deployment auf Hetzner. Zuerst bauen, damit ab dem ersten Feature auf dem echten iPhone getestet wird.
1. **Stilprofil anlegen/bearbeiten** inkl. 5–10 Inspirationsbilder und **Frisur & Bart** (Porträtfoto + Attribute). Frisur/Bart lassen sich jederzeit mit einem Tipp aktualisieren ("war beim Friseur"), dabei entsteht ein neuer `grooming_state`-Eintrag.
2. **Teil erfassen:** Foto hochladen oder aus dem Internet importieren → LLM schlägt Kategorie, Farben, Muster, Material, Textur vor → Nutzer bestätigt/korrigiert und ergänzt Passform, Länge/Schnitt, Rolle, Formalität, Wärme, Tragevarianten. Die Eingabe muss auf dem iPhone schnell gehen (Auswahl-Chips statt Textfelder, wo möglich).
3. **Schrank-Ansicht:** Raster aller Teile, filterbar nach Kategorie.
4. **Outfit-Generator:** Eingabe Anlass (Ort, Leute, Vibe) + Wetter → 3 Outfits mit Begründung und Styling-Moves.
5. **Outfits speichern** mit Namen.
6. **Feedback:** Daumen hoch/runter + optionaler Kommentar. Die letzten ~10 Feedbacks gehen bei jeder Generierung als Beispiele mit.

### Stufe 2 – nach dem ersten Wochenende
7. **Spiegel-Check:** Selfie im Outfit hochladen → konkretes Feedback zu Passform und Styling-Moves. Erkannte Abweichungen (z. B. "Hose sitzt weiter als angegeben") als Korrekturvorschlag für das Teil anbieten.
8. **Hero-Piece-Modus:** "3 Outfits um Teil X".
9. **Kandidaten-Check ("Passt das zu meinem Schrank?"):** Produktfoto oder Produkt-URL eines Teils, das noch nicht gekauft ist → mit wie vielen Teilen kombinierbar + 3 Beispiel-Outfits. Kandidat wird NICHT als Item gespeichert (erst nach Kauf, dann mit einem Tipp übernehmbar).
10. **Kalender / Trage-Historie:** Outfit als getragen markieren; Kalenderansicht, was wann getragen wurde; Generator vermeidet kürzlich getragene Kombis.
11. **Swipe-Builder + Outfit-Score:** Outfit selbst zusammenstellen, indem man pro Slot (Oberteil, Unterteil, Schuhe, optional Jacke/Layer/Accessoire) horizontal durch die passenden Teile wischt. Auf Knopfdruck bewertet das LLM das Outfit ehrlich (Score 1–10 + Begründung + max. 3 konkrete Verbesserungen, z. B. Styling-Move oder Tausch eines Teils aus dem Schrank). Verbesserungsvorschläge lassen sich mit einem Tipp übernehmen (Teil wird im Builder getauscht). Speichern mit `quelle = swipe`.

### Stufe 3 – nur wenn Stufe 1+2 wirklich genutzt werden
12. **Lückenanalyse:** Welches eine Teil/Accessoire würde die meisten neuen starken Outfits ermöglichen? Ergebnis als präzise Beschreibung + generierte Suchlinks (Zalando, About You). Keine Shop-Integration.

### Stufe 4 – native iOS-App (nur wenn Stylomat nach 2–4 Wochen noch regelmäßig genutzt wird)
13. **SwiftUI-App als zweiter Client** auf dieselbe API, inkl. Token-Login in `auth.py`.
14. **Teilen-Menü-Erweiterung:** In Safari (Produktseite) oder Fotos-App "Teilen → Stylomat" → Import bzw. Kandidaten-Check.
15. **Automatisches Freistellen auf dem Gerät** über das iOS-Vision-Framework vor dem Upload.
16. **Swipe-Builder nativ** mit flüssigen Animationen und haptischem Feedback.

## Ausdrücklich NICHT bauen (Scope-Schutz)

- Keine eigenen Machine-Learning-Modelle / kein Training
- Kein virtuelles Anprobieren, keine generierten Outfit-Bilder, keine Collagen
- Keine Registrierung, kein Mehrnutzerbetrieb (genau ein Account, per Kommandozeile angelegt)
- Keine Shop-Anbindung; beim URL-Import wird nur das Hauptbild geladen, sonst kein Scraping
- Keine Social-Features
- Keine eigene Frisur-/Bartberatung als Feature (nur kurze Hinweise im Spiegel-Check)
- Keine automatische Hintergrund-Entfernung auf dem Server in Stufe 1
- Kein Kalender-/Wetter-API-Anschluss (Wetter wird eingetippt)
- Keine native iOS-App vor Stufe 4 – aber jede Entscheidung in Stufe 1–3 muss sie möglich halten (API-first)

## Prompts (erste Fassung, liegen unter `prompts/`)

### `prompts/tagging.md` – Teil erkennen
Du bekommst ein Foto eines Kleidungsstücks. Schlage vor: kategorie, unterkategorie, farben, muster, material,
textur. Gib bei Unsicherheit "unsicher" an statt zu raten. Passform, Rolle, Formalität und Wärme NICHT raten –
die trägt der Nutzer ein. Wenn auf dem Bild ein Model das Teil trägt (typisch bei Shop-Bildern), beschreibe nur
das Kleidungsstück, nicht das restliche Outfit. Antworte als JSON.

### `prompts/outfit_generator.md` – Outfits erzeugen
Du bist ein ehrlicher, stilsicherer Menswear-Stylist. Ziel: Outfits, die richtig stark aussehen, nicht nur korrekt.

Du bekommst:
- das Stilprofil (Statur, Proportionen, Farbtyp, Gesichtsform, Vibe-Mix, Lieblingsfarben, No-Gos) und Inspirationsbilder
- den aktuellen Frisur-/Bart-Stand inkl. Porträtfoto
- alle aktiven Teile mit Foto und allen Attributen
- Anlass (Ort, Leute, Vibe) und Wetter
- die letzten Feedbacks inkl. Kommentaren
- kürzlich getragene Outfits (vermeiden)

Regeln:
- Nur Teile aus dem Schrank verwenden, über ihre ID referenzieren.
- Pro Outfit genau ein Fokuspunkt.
- Passform und Proportionen zuerst prüfen: Volumen oben vs. unten, Hosenweite zu Schuhvolumen, Länge der Oberteile.
- Farben nah am Gesicht passend zum Farbtyp sowie zu Haar- und Bartfarbe.
- Kragen/Halsbereich auf Bart abstimmen (voller Bart → eher offene Kragen; kurzer Bart/rasiert → Rollkragen möglich).
- Frisurvolumen bei den Proportionen mitdenken.
- Texturkontrast aktiv nutzen.
- Zu jedem Outfit konkrete Styling-Moves angeben (krempeln, einstecken, offen tragen, Layering).
- Die 3 Outfits sollen sich deutlich unterscheiden (nicht dreimal dieselbe Hose).
- Wenn der Schrank für den Anlass nichts Starkes hergibt, das ehrlich sagen und das fehlende Teil benennen.

Ausgabe als JSON: pro Outfit item_ids, name, begruendung (2–4 Sätze, warum es funktioniert),
styling_moves, fokuspunkt.

### `prompts/outfit_score.md` – Selbst gebautes Outfit bewerten
Du bekommst ein vom Nutzer zusammengestelltes Outfit (Fotos + Attribute aller Teile), das Stilprofil,
den aktuellen Frisur-/Bart-Stand, optional den Anlass und den restlichen Schrank. Bewerte ehrlich auf einer
Skala von 1–10, wie stark das Outfit ist – nicht nur, ob es "korrekt" ist. Prüfe: ein Fokuspunkt,
Silhouette/Proportionen, Hosenweite zu Schuhvolumen, Farben nah am Gesicht, Texturkontrast, Anlass.
Gib eine kurze Begründung und maximal 3 konkrete Verbesserungen (Styling-Move oder Tausch gegen ein Teil aus
dem Schrank, per ID). Kein Schönreden: Ein mittelmäßiges Outfit bekommt eine mittelmäßige Zahl. Antworte als JSON.

### `prompts/mirror_check.md` – Spiegel-Check
Du siehst den Nutzer im Outfit. Beurteile ehrlich und konkret: Sitz an Schultern, Taille, Beinen, Länge,
Proportionen, Zusammenspiel Hose/Schuh. Gib maximal 3 konkrete Verbesserungen (Styling-Moves oder Tausch
eines Teils aus dem Schrank). Wenn ein Teil offensichtlich anders sitzt als im Attribut angegeben, schlage
eine Korrektur des Attributs vor. Frisur und Bart gehören zum Gesamtbild: gelegentlich ein kurzer Hinweis ist
erlaubt (z. B. "zu diesem Look würde ein kürzer getrimmter Bart cleaner wirken"), aber keine ausführliche
Frisur-/Bartberatung. Keine Komplimente ohne Begründung.

## Erfolgskriterien (nach 4 Wochen prüfen)

1. Richi nutzt die App mindestens 3× pro Woche.
2. Mindestens einige getragene Outfits, auf die er allein nicht gekommen wäre.
3. Freunde fragen von selbst nach der App.

Erst wenn alle drei erfüllt sind: kleine Beta mit 3–5 Freunden, dann Datenschutz (DSGVO), Registrierung und Kosten klären.

## Noch offen

- Hostname (eigene Subdomain oder vorerst sslip.io)
- Optional vor dem Bauen: Fits mit ~10 Teilen als Benchmark testen
