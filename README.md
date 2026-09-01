> Hinweis: Die Notebooks sind als Vorlagen und zur Inspiration gedacht, die zur Weiterbearbeitung anregen sollen.


# Differenzierte Zinsen – Notebook 1

## Über dieses Notebook

Das Notebook und das dazugehörige Datenset befinden sich im GitHub-Repositorium unter folgendem Link:
[economies-of-space-proseminar-hs2026/differenzierte_zinsen](https://github.com/history-unibas/economies-of-space-proseminar-hs2026/tree/main/differenzierte_zinsen)

Das Notebook läuft mit `Python 3.12.6` (wurde ausschliesslich mit dieser Version getestet).

## Datenset

Das Datenset dieses Notebooks enthält historische Informationen zu Zinsarten, Zinshöhen und Zinsempfänger:innen. Darüber hinaus enthält es verschiedene Metadaten und weitere Informationen – so zum Beispiel georeferenzierte Adressen. Die Texte, aus denen die Informationen stammen, werden in den Dataframes stets reproduziert.

Für ein tieferes Verständnis der Quellen und des Datensatzes sollten die Repositorien und Papers von Ökonomien des Raums konsultiert werden (Repositorium: economies of space / Webpage: Ökonomien des Raums / Dissertation: Vonwiller 2026).

Das in diesem Notebook bearbeitete Datenset `rentsall_korrektur_df.csv` wurde aus mehreren Datensets zusammengesetzt, die mir Ismail Prada bereitgestellt hat:

- ein Datenset, das überwiegend auf den Zinsbeschreibungen basiert, die unterhalb der Hauskaufurkunden dokumentiert wurden
- ein Datenset, das Informationen zu den Renten direkt aus den Rentenkaufurkunden entzieht

Das entspricht zwei verschiedenen Arten von Urkunden: Ersteres ist ein Nebenprodukt, das zur Beschreibung von Zinsen beim Verkauf einer Liegenschaft entstanden ist, zweiteres die Transaktion der Renten selbst.

Darüber hinaus habe ich die Spalten von `rentsall_korrektur_df.csv` den Spalten eines weiteren Datensets mit dem Titel `froenfuerzins_korrektur_df.csv` angeglichen, um Analysen, die beide Datensets verbinden, zu erleichtern.

## Mitwirkende

An der Genese der Daten haben folgende Personen mitgearbeitet:

- Benjamin Hitz
- Ismail Prada
- Katrin Fuchs
- Jonas Aeby
- Aline Vonwiller

## Änderungen an den Ursprungsdaten

Um die beiden Datensets von Ismail Prada zusammensetzen zu können, mussten zur Vorbereitung Änderungen daran vorgenommen werden. Diese Änderungen sind im Notebook selbst nicht mehr nachvollziehbar. Aus diesem Grund folgt hier eine kurze Erläuterung der wichtigsten Anpassungen, die zum aktuellen Datensatz `rentsall_korrektur_df` geführt haben:

### Umwandlung der `year`-Spalte in Float

```python
# Umwandeln der 'year' Spalte in Float
rents_all_df['year'] = rents_all_df['year'].astype(float)

# Überprüfe den neuen Datentyp
neuer_datentyp = rents_all_df['year'].dtype
print(neuer_datentyp)
```

### Reduktion von Duplikaten

Beim Zusammensetzen der beiden Datensets entstanden Duplikate – zum Beispiel ein und dieselben Renteninformationen, die sowohl aus Zinsbeschreibungen als auch aus Rentenverkäufen ermittelt wurden. Diese wurden mit folgendem Code reduziert:

```python
renten_zusätzlich = renten_zusätzlich.drop_duplicates(
    subset=['dossier', 'year', 'cause'],
    keep='first'
)
```

### Neue Spalte für Personen

Bis dahin bestand nur eine separate Spalte für Institutionen, aber keine für Personen. Die Spalte `beneficiary` enthielt alle Akteursgruppen gemeinsam. Daher wurde eine eigene Personenspalte generiert, basierend darauf, ob die Spalte `beneficiary_org_norm` einen Treffer enthält:

```python
# Nach Erstellung des DFs, weitere Bereinigungen:
# Eine eindeutige Personenspalte hinzufügen, die anhand davon generiert wird,
# dass die beneficiary_org_norm-Spalte keinen Treffer beinhaltet

def fill_logic(row):
    # Prüft: Ist der Wert in 'beneficiary_org_norm' leer oder NaN?
    if pd.isna(row['beneficiary_org_norm']) or row['beneficiary_org_norm'] == '':
        # Wenn ja → gib den Wert aus 'beneficiary' zurück
        return row['beneficiary']
    else:
        # Wenn nein → gib einen leeren String zurück
        return ""

# Wendet die Funktion zeilenweise auf den DataFrame an
renten_zusätzlich['beneficiary_per'] = renten_zusätzlich.apply(fill_logic, axis=1)
```

### Vereinheitlichung von Spaltennamen

Um das Zusammensetzen der Datensets zu erleichtern, wurden Spaltennamen vereinheitlicht, zum Beispiel die Institutionsspalte:

```python
renten_zusätzlich.rename(columns={'beneficiary_org_norm': 'claimantOrg'}, inplace=True)
```

### Verschiebung von Treffern in die Personenspalte

Orientierungspunkt ist die Spalte `claimantOrg`: Immer wenn diese den Wert `no_org` enthält, wird der entsprechende Treffer aus der Spalte `beneficiary` (die alle Zinsempfänger:innen enthält, unabhängig davon, ob es sich um eine Institution oder eine Person handelt) in die neue Spalte `beneficiary_per` kopiert und anschliessend aus `beneficiary` gelöscht:

```python
# 1. Definieren, welche Zeilen betroffen sind
mask = rents_all_df['claimantOrg'] == 'no_org'

# 2. Nur in diesen Zeilen: Kopiere von beneficiary nach beneficiary_per
# Bestehende Werte in beneficiary_per (wo claimantOrg NICHT 'no_org' ist) bleiben erhalten!
rents_all_df.loc[mask, 'beneficiary_per'] = rents_all_df.loc[mask, 'beneficiary']

# 3. Nur in diesen Zeilen: Lösche den Wert in beneficiary
rents_all_df.loc[mask, 'beneficiary'] = None
```

### Manuelle Nachbearbeitung unbekannter Institutionen

Einige Institutionen waren automatisch als `unk` (unknown) markiert, da sie nicht erkannt wurden. Diese wurden manuell nachbearbeitet und wieder hochgeladen, um die Korrekturen ins Datenset zurückzuführen. Überall, wo die Spalte `korrektur` einen Wert enthält, wird dieser bei `claimantOrg` an der richtigen Stelle eingesetzt. Anschliessend wird die Spalte `korrektur` wieder entfernt:

```python
rents_all_df_corrected = rents_all_df.merge(
    unk_org_tbc_korrektur[['beneficiary', 'id', 'korrektur']],
    on=['beneficiary', 'id'],
    how='left'
)

rents_all_df_corrected['claimantOrg'] = rents_all_df_corrected['korrektur'].where(
    rents_all_df_corrected['korrektur'].notna(),
    rents_all_df_corrected['claimantOrg']
)

# Der folgende Schritt kann entfernt werden, falls kontrolliert werden soll,
# ob die Korrekturen richtig ersetzt wurden
rents_all_df_corrected.drop(columns=['korrektur'], inplace=True)

rents_all_df_corrected
```

### Speicherung des finalen Datensets

```python
rents_all_df_corrected.to_csv('rentsall_korrektur_df.csv', index=False)
```


# Beliebige Institutionen: Zinsen & Frönungen – Notebook 2

## Über dieses Notebook

Dieses Notebook (`NB2_beliebige_insti_zinsen_froenen`) baut auf dem Datenset `rentsall_korrektur_df.csv` aus Notebook 1 auf und ergänzt es um ein zweites Datenset: `froenungenfuerzins_korrektur_df.csv`.

## Datenset

Frönungen bezeichnen Klagen, die im Falle von immobilienbezogenen Schulden beim Schultheissengericht eingereicht wurden.

Die Treffer im Frönungen-Datenset entsprechen einzelnen Ereignissen – jeweils einem konkreten Zeitpunkt, an dem eine Klage eingereicht und dabei eine Institution als klagende Partei genannt wurde. Das unterscheidet sich grundlegend von der Datengrundlage der Zinsarten in Notebook 1: Bei den Zinsarten (`rentsall_korrektur_df.csv`) sind die erfassten Informationen eher ein indirektes Nebenprodukt, da die zugrunde liegenden Urkunden primär Hauskäufe dokumentieren und nicht exakte Zinsmomente. Bei den Frönungen (`froenungenfuerzins_korrektur_df.csv`) hingegen bildet jeder Treffer tatsächlich ein konkretes Klage-Ereignis ab.

Anders als bei den Zinsarten müssen die Treffer der Frönungen daher nicht pro Dossier aggregiert werden, um den Hauskaufeffekt herauszufiltern.

Der genaue Anlass der jeweiligen Klage (zum Beispiel ein Drittkauf) ist im Datenset nicht identifiziert. Auch die genannten Klagegründe sind bislang nicht in einer separaten Spalte extrahiert. Dies sind nur zwei von mehreren möglichen Ansatzpunkten zur Weiterverarbeitung dieses Datensets.

## Qualität der Organisationszuordnung

Die automatisiert erkannten Organisationen wurden für den Zeitraum 1400–1530 zusätzlich manuell kontrolliert und korrigiert, sofern die automatische Erkennung fehlerhaft war.

Der Johanniterorden in der St. Johanns-Vorstadt und die St. Johanns-Bruderschaft auf dem Münster sind im Datenset noch nicht sauber voneinander getrennt.

Institutionen, die nicht Gegenstand der zugrunde liegenden Dissertation waren, wurden nicht mit derselben Sorgfalt geprüft. Bei diesen empfiehlt sich eine erneute manuelle Kontrolle.

## Hinweis zur Nutzung des Codes

Die meisten Codeblöcke in diesem Notebook sind so aufgebaut, dass der Name der jeweiligen Institution in der ersten Zeile ausgetauscht werden kann, um das Notebook auf eine andere Institution anzuwenden.
