# MCP Server Fix: connect_to_net persistiert keine Netze

## Datum: 2026-03-31

## Problem

`connect_to_net` und `add_net` melden Erfolg, aber nach `save_project` sind keine Netze in der `.kicad_pcb`-Datei vorhanden. Nur `(net 0 "")` bleibt übrig. Ratsnest-Linien werden in KiCad nicht angezeigt.

## Root Cause (2 Teile)

### 1. `connect_to_net` ist nur Schematic-seitig

`_handle_connect_to_net` in `kicad_interface.py` hat nur das Schematic bearbeitet — Wire-Stubs und Net-Labels über `ConnectionManager.connect_to_net()` hinzugefügt. Die PCB-Pads (`pad.SetNet()`) wurden **nicht** gesetzt. Dadurch referenziert kein Board-Element das Netz.

### 2. `pcbnew.SaveBoard()` löscht Orphan-Netze

KiCad's `SaveBoard()` schreibt nur Netze in die Datei, die von mindestens einem Board-Element (Pad, Track, Via, Zone) referenziert werden. Netze ohne Referenzen werden beim Speichern verworfen.

### Kombinierter Effekt

1. `add_net` erstellt Netz im Speicher ✓
2. `connect_to_net` ändert nur Schematic, nicht PCB-Pads → Netz bleibt "orphan" auf dem Board
3. `save_project` → `pcbnew.SaveBoard()` löscht alle Orphan-Netze
4. Ergebnis: nur `(net 0 "")` in der gespeicherten Datei

## Fix

### Datei: `python/kicad_interface.py`

#### 1. Neue Hilfsmethode `_assign_net_to_pad`

Erstellt das Netz auf dem Board falls nötig, sucht den Footprint per Reference und das Pad per Nummer, und ruft `pad.SetNet()` auf.

#### 2. `_handle_connect_to_net` erweitert

Nach der Schematic-Operation wird jetzt zusätzlich `_assign_net_to_pad()` aufgerufen, um das Netz auch dem PCB-Pad zuzuweisen.

#### 3. `_handle_connect_passthrough` erweitert

Gleicher Fix — nach der Schematic-Passthrough-Verbindung werden die Netze auch den PCB-Pads zugewiesen.

#### 4. `_BOARD_MUTATING_COMMANDS` erweitert

`"connect_to_net"` hinzugefügt, damit Auto-Save nach der PCB-Pad-Zuweisung greift.

## Verifikation

Nach dem Fix:
- `connect_to_net` setzt `pad.SetNet()` auf dem Board
- `save_project` / Auto-Save persistiert die Netze korrekt
- Ratsnest-Linien erscheinen in KiCad

---

# Weitere bekannte MCP-Probleme (Stand 2026-03-31)

## place_component Duplikate

### Problem
`place_component` erstellt Footprints auf dem Board, aber wenn das Board vorher schon Footprints mit gleicher Reference enthält (z.B. nach fehlgeschlagenem Reset), entstehen Duplikate. `get_component_list` gibt bei Duplikat-References keine Ergebnisse zurück.

### Workaround
Vor dem Platzieren sicherstellen, dass die PCB-Datei sauber ist (keine existierenden Footprints mit gleicher Reference). Bei Bedarf: PCB-Datei manuell auf leeren Zustand zurücksetzen, `/mcp` reconnecten, `open_project` aufrufen.

## create_project setzt In-Memory Board nicht zurück

### Problem
`create_project` erstellt neue Dateien auf Disk, aber der MCP-Server behält das alte Board im Speicher. Nachfolgende `place_component`-Aufrufe arbeiten auf dem alten Board.

### Workaround
Nach `create_project` immer `/mcp` reconnecten und dann `open_project` aufrufen, damit die neue leere Datei geladen wird.

## place_component Rotation auf B.Cu

### Problem
`place_component` mit `rotation: 180` auf `B.Cu` wird manchmal nicht korrekt übernommen — die Component-List zeigt rotation 0.

### Workaround
Erst mit rotation 0 platzieren, dann `rotate_component` mit angle 180 aufrufen. Anschließend Pad-Positionen mit `get_component_pads` verifizieren.

## add_zone nicht verfügbar

### Problem
`add_zone` wird als "Unknown command" zurückgewiesen.

### Workaround
`add_copper_pour` verwenden — hat ähnliche Funktionalität (Layer, Net, Outline, Clearance).

## Kein Tool für Keep-Out-Zonen

### Problem
Es gibt kein MCP-Tool für Rule Areas / Keep-Out-Zonen (z.B. Antennen-Freihaltebereich).

### Workaround
Copper Pour mit angepasstem Outline verwenden, das den Freihaltebereich ausspart. Für echte Keep-Out-Zonen muss die PCB-Datei direkt editiert werden (nach Rückfrage beim User).

## create_netclass: `netclasses_map` hat kein `Find`

### Problem (2026-04-01)
`create_netclass` schlägt fehl mit `'netclasses_map' object has no attribute 'Find'`. KiCad 8+ hat die Netclass-API geändert — `board.GetNetClasses()` gibt eine `netclasses_map` (std::map-Wrapper) zurück, die nur `find` (lowercase), `has_key`, etc. hat, aber kein `Find()` und kein `Add()`.

### Fix
**Datei:** `python/commands/routing.py`, Funktion `create_netclass`

Die alte API (`board.GetNetClasses().Find()/.Add()`) durch die neue `NET_SETTINGS`-API ersetzt:
```python
ds = self.board.GetDesignSettings()
ns = ds.m_NetSettings
netclass = pcbnew.NETCLASS(name)
# ... SetTrackWidth, SetClearance etc. ...
ns.SetNetclass(name, netclass)
# Netz-Zuordnung via Pattern:
ns.SetNetclassPatternAssignment(name, net_name)
```

Außerdem uVia-Getter gefixt: `GetMicroViaDiameter()` → `GetuViaDiameter()`, `GetMicroViaDrill()` → `GetuViaDrill()`.

### Wichtig
`SetNetclassPatternAssignment(netclass, pattern)` speichert in der `.kicad_pro`-Datei, nicht in `.kicad_pcb`. Die Zuordnung wird beim Projekt-Öffnen geladen.
