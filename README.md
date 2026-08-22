# 🏭 Industrial IoT Bridge — integracja z Odoo 17 MRP

![Python](https://img.shields.io/badge/Python-3.10%2B-blue)
![Modbus](https://img.shields.io/badge/Modbus-TCP-red)
![Odoo](https://img.shields.io/badge/Odoo-17.0%20MRP-purple)
![License](https://img.shields.io/badge/License-MIT-green.svg)
![Status](https://img.shields.io/badge/status-demo%20%2F%20portfolio-orange)

**Opracowane przez [Fayna Digital](https://fayna.agency) — Autor: Volodymyr Shevchenko**

---

## Problem

Drukarnie i zakłady produkcyjne z 2–10 maszynami przemysłowymi (offset,
digital, CNC) śledzą produkcję tak, jak większość MŚP: operator chodzi po
hali z kartką i na koniec zmiany przepisuje liczniki do ERP. To wolne,
podatne na błędy, a zanim kierownik zobaczy liczby, zmiana już się kończy.

## Rozwiązanie

Lekki **Industrial IoT Bridge** napisany w Pythonie, który odczytuje stan
maszyn bezpośrednio z hali produkcyjnej — przez Modbus TCP — i przesyła go
prosto do **zleceń roboczych Odoo 17 MRP** przez XML-RPC, z lokalnym buforem
SQLite, dzięki czemu żaden odczyt nie ginie, gdy Odoo jest chwilowo
nieosiągalne. Dołączony symulator Modbus (`plc_simulator.py`) pozwala
przeprowadzić cały przepływ end-to-end bez fizycznego sprzętu — i właśnie z
tym startuje to publiczne demo.

> **Uwaga:** to projekt portfolio/demo. Logika specyficzna dla klienta,
> dane uwierzytelniające i mapy rejestrów zostały wykluczone — zobacz
> [Kontekst biznesowy](#kontekst-biznesowy).

## Wynik

- Ręczne wpisywanie liczników wyeliminowane — dane produkcyjne trafiają do
  zleceń roboczych Odoo na bieżąco, a nie na koniec zmiany.
- Lokalne buforowanie oznacza, że niestabilne łącze sieciowe nie oznacza już
  utraty danych: bridge dalej odczytuje, kolejkuje lokalnie i synchronizuje,
  gdy tylko Odoo znów jest osiągalne.
- Potwierdzona tu koncepcja działa obecnie w produkcji, rozszerzona do pełnego
  modułu Odoo z kioskiem operatora i panelem menedżera — zobacz
  [Projekty powiązane](#projekty-powiazane).

## Stack

Python 3.10+ · [pymodbus](https://github.com/pymodbus-dev/pymodbus) (Modbus
TCP) · `xmlrpc.client` (integracja z Odoo, stdlib) · SQLite (lokalny bufor) ·
pytest / ruff / mypy.

---

## Architektura

```
┌─────────────────────────────────────────────────┐
│             Hala maszyn przemysłowych             │
│   Heidelberg XL-106  │  HP Indigo 6K             │
│   (druk offsetowy)   │  (druk cyfrowy)           │
└────────────┬─────────────────────────────────────┘
             │  Modbus TCP / heartbeat ICMP
             ▼
┌─────────────────────────────────────────────────┐
│             Industrial IoT Bridge                │
│  machine_tester.py  ← heartbeat / łączność       │
│  scanner.py         ← czytnik Modbus TCP         │
│  main.py            ← orkiestrator               │
│  odoo_api.py        ← klient XML-RPC do Odoo     │
│  db_handler.py      ← lokalny bufor (SQLite)     │
└────────────┬─────────────────────────────────────┘
             │  Odoo XML-RPC
             ▼
┌─────────────────────────────────────────────────┐
│              Odoo 17 MRP                         │
│  Zlecenia robocze  │  Raporty produkcyjne  │ MO  │
└─────────────────────────────────────────────────┘
```

## Komponenty

| Plik | Rola |
|------|------|
| `src/bridge/main.py` | Orkiestrator — sprawdza każdy monitorowany zasób, prowadzi pętlę odczytu |
| `src/bridge/machine_tester.py` | Heartbeat łączności — symulacja w trybie demo lub prawdziwy ping ICMP |
| `src/bridge/scanner.py` | Czytnik Modbus TCP — odczytuje rejestry trzymające (status, prędkość, licznik) |
| `src/bridge/odoo_api.py` | Klient Odoo XML-RPC — przesyła odczyty do zleceń roboczych |
| `src/bridge/db_handler.py` | Lokalny bufor SQLite — kolejkuje dane, gdy Odoo jest nieosiągalne |
| `src/bridge/plc_simulator.py` | Symulator serwera Modbus — pełny przepływ bez fizycznego sprzętu |
| `config/settings.py` | Cała konfiguracja sterowana zmiennymi środowiskowymi — bez sekretów w kodzie |

## Mapa rejestrów Modbus

| Rejestr | Wartość |
|----------|-------|
| `HR[0]` | Status: `0` = bezczynny, `1` = pracuje |
| `HR[1]` | Prędkość (jednostki/godz.) |
| `HR[2]` | Licznik (łączna liczba wyprodukowanych jednostek) |

> Mapy rejestrów różnią się w zależności od modelu maszyny — skonfiguruj w
> `src/bridge/scanner.py` → `read_machine_state()`.

---

## Szybki start

### Wymagania

- Python 3.10+
- `pip install -r requirements.txt`
- Instancja Odoo 17.0 (lub pomiń ją — strona Modbus działa w pełni offline)

### Uruchomienie z dołączonym symulatorem (bez sprzętu, bez Odoo)

```bash
# Terminal 1 — uruchom symulator Modbus
python -m src.bridge.plc_simulator
# Udostępnia serwer Modbus na 127.0.0.1:5020

# Terminal 2 — uruchom bridge (tryb demo, tylko sprawdzenie łączności)
python -m src.bridge.main
```

### Konfiguracja

Wszystkie ustawienia to zmienne środowiskowe (pełna lista i wartości domyślne
w `config/settings.py`) — nic nie jest zaszyte na sztywno:

```bash
export BRIDGE_ODOO_URL="https://twoja-instancja-odoo.com"
export BRIDGE_ODOO_DB="twoja_baza"
export BRIDGE_ODOO_USER="admin@firma.com"
export BRIDGE_ODOO_PASSWORD="twoj_klucz_api"
export BRIDGE_DEMO_MODE=false   # użyj prawdziwego heartbeat ICMP zamiast symulacji
```

### Testy

```bash
python -m pytest tests/ -v
```

---

## Kontekst biznesowy

**Branża:** poligrafia / produkcja
**Profil klienta:** drukarnia lub zakład z 2–10 maszynami przemysłowymi
**Rozwiązany problem:** ręczne wprowadzanie danych produkcyjnych do ERP —
wolne, podatne na błędy, opóźnione o całą zmianę
**Rozwiązanie:** zautomatyzowany bridge przesyłający wyniki maszyn w czasie
rzeczywistym prosto do zleceń roboczych Odoo
**Przetwarzanie danych:** wszystkie dane pozostają on-premises lub w chmurze
klienta — bez zewnętrznego SaaS w pętli; bridge sam nie generuje ani nie
przechowuje danych osobowych

## Projekty powiązane

- [fayna-shopfloor-kiosk](https://github.com/fayna-digital/fayna-shopfloor-kiosk) — pełny
  produkcyjny moduł Odoo, do którego rozszerzono koncepcję tego demo: kiosk
  operatora, panel menedżera i utwardzony bridge Modbus.

## Licencja

MIT — zobacz [LICENSE](LICENSE).

---

*Zbudowane przez [Fayna Digital](https://fayna.agency) · Volodymyr Shevchenko*
*Architektura systemów i automatyzacja przemysłowa dla produkcyjnych MŚP*
