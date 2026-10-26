# Fermabets

Μοντέλο για player props σε Euroleague και NBA. Διαβάζει τα στοιχεία από Google Sheet, κατεβάζει το ιστορικό του παίκτη, βγάζει πιθανότητα, EV και ποντάρισμα, και τα γράφει πίσω στο ίδιο φύλλο.

| Πρωτάθλημα | Καρτέλα | Ιστορικό παίκτη | Pace | Εντολή |
| --- | --- | --- | --- | --- |
| Euroleague | `Daily Bets Inputs` | Hackastat | Hackastat | `python run_daily_bets.py --cdp http://127.0.0.1:9222` |
| NBA | `NBA Daily` | Basketball-Reference | pbpstats | `python run_nba_daily.py` |

Οι δύο ροές τρέχουν ξεχωριστά. Η μία δεν διαβάζει ούτε γράφει την καρτέλα της άλλης.

## Εγκατάσταση

```powershell
pip install -r requirements.txt
python -m playwright install chromium
```

Το κλειδί του Google δεν είναι στο repo. Βάλε το δικό σου αρχείο service account ως `service_account.json` στον φάκελο (υπόδειγμα: `service_account.example.json`) και κάνε share το Google Sheet ως Editor στο `client_email` του αρχείου.

Το Google Sheet ορίζεται στο `config.py` ή με τη μεταβλητή `FERMABETS_SHEET_ID`.

## Euroleague

Το Hackastat έχει Cloudflare, οπότε το script συνδέεται σε κανονικό Chrome που ανοίγεις εσύ.

1. Κλείσε όποιο Chrome χρησιμοποιεί το προφίλ `.hackastat_chrome`.
2. Άνοιξε Chrome με port:

```powershell
cd "$env:USERPROFILE\OneDrive\Desktop\Fermabets"
& "C:\Program Files\Google\Chrome\Application\chrome.exe" --remote-debugging-port=9222 --user-data-dir="$PWD\.hackastat_chrome"
```

3. Άνοιξε το hackastat.eu, πέρασε το Verify αν ζητηθεί και άφησε το Chrome ανοιχτό.
4. Σε δεύτερο terminal:

```powershell
python run_daily_bets.py --cdp http://127.0.0.1:9222
```

## NBA

Δεν χρειάζεται Chrome ούτε port.

```powershell
python run_nba_daily.py
```

- Στο κελί του παίκτη βάζεις ένα ή δύο links gamelog του Basketball-Reference, ένα για κάθε σεζόν. Το `/gamelog/2026/` είναι η σεζόν 2025-26. Το usage διαβάζεται αυτόματα από την advanced σελίδα της ίδιας σεζόν.
- Στο `Exp Pace` γράφεις το εύρος και, σε παρένθεση, το link του pbpstats για την ομάδα του παίκτη:

```text
98-102 (https://www.pbpstats.com/game-logs/nba/team?Season=2026-27,2025-26&SeasonType=All&EntityId=1610612765)
```

Το pace μπαίνει σε έναν αγώνα μόνο όταν ταιριάζουν και η ημερομηνία και ο αντίπαλος.

## Επιλογές

```powershell
python run_daily_bets.py --cdp http://127.0.0.1:9222 --dry-run
python run_nba_daily.py --player "Durant"
```

- `--dry-run`: τρέχει το μοντέλο χωρίς να γράψει στο φύλλο.
- `--player`: τρέχει μόνο όσους παίκτες περιέχουν αυτό το κομμάτι ονόματος.

## Πώς γράφεις τα στοιχεία

| Τι γράφεις | Τι σημαίνει |
| --- | --- |
| `21` | 21 και πάνω |
| `20-25` | από 20 έως 25 |
| `21!` ή `!21` | ακριβώς 21 |

Ισχύει για λεπτά, FGA, FTA, usage, pace και 3PA. Ένα κενό κελί βγαίνει από το μοντέλο και τα υπόλοιπα βάρη μοιράζονται ξανά στο 100%.

- **Is Home?**: έδρα ή εκτός.
- **H2H Opponent**: ο αντίπαλος. Οι αγώνες απέναντί του μετράνε μόνο στο H2H και βγαίνουν από τα υπόλοιπα φίλτρα.
- Γραμμή με κενό `Model Prob (%)` θεωρείται εκκρεμής και τρέχει.
- Γραμμή με `Status` που ξεκινά με `ERROR` προσπερνιέται. Σβήσε το Status για να ξανατρέξει.
- Όταν λείπει το link του παίκτη, χρησιμοποιείται το τελευταίο link από παλιότερη γραμμή με ακριβώς το ίδιο όνομα.

## Μοντέλο

Η πιθανότητα βγαίνει από δύο στρώσεις:

- **Φίλτρα (55%)**: ποσοστό επιτυχίας στους αγώνες που ταιριάζουν σε κάθε στοιχείο, με αξιοπιστία Bühlmann `N/(N+5)`. Για το H2H, `N/(N+2)`.
- **Ομοιότητα (45%)**: κάθε παλιός αγώνας ζυγίζεται ανάλογα με το πόσο μοιάζει με τη σημερινή πρόβλεψη.

Βάρη ανά στατιστικό:

| Στατιστικό | Βάρη |
| --- | --- |
| Πόντοι | λεπτά 30%, FGA 25%, FTA 10%, 3PA 10%, usage 15%, pace 10%, έδρα 10%, H2H 10% |
| Τρίποντα | 3PA 25%, λεπτά 20%, FGA 15%, volume (usage και pace) 15%, έδρα 15%, H2H 10% |
| Ριμπάουντ | λεπτά 40%, pace 25%, έδρα 20%, H2H 15% |
| Ασίστ | λεπτά 30%, pace 20%, FGA 15%, usage 15%, έδρα 10%, H2H 10% |

Ποντάρισμα: quarter-Kelly με ανώτατο όριο 5% του bankroll (5€ στα 100€). EV = πιθανότητα × απόδοση − 1. Με αρνητικό EV, γράφει NO BET.

## Τι γράφει στο φύλλο

`Model Prob (%)`, `EV (%)`, `Stake (€)`, `Status` και `Post`. Το `Post` είναι ένα σύντομο κείμενο για δημοσίευση και γράφεται μόνο όταν το ποντάρισμα είναι στο ανώτατο όριο. Τα `Result` και `Profit/Loss (€)` δεν τα αγγίζει ποτέ.

## Αρχεία

| Αρχείο | Ρόλος |
| --- | --- |
| `run_daily_bets.py` | Euroleague |
| `run_nba_daily.py` | NBA |
| `props_model.py` | φίλτρα, ομοιότητα, Kelly, κείμενο Post |
| `hackastat_scraper.py` | ιστορικό Euroleague από Hackastat |
| `nba_sources.py` | Basketball-Reference και pace από pbpstats |
| `sheets_client.py` | ανάγνωση και εγγραφή στο Google Sheet |
| `config.py` | βάρη, Sheet ID, στήλες |
| `euroleague_player_props.py`, `google_sheets_bridge.gs` | παλιό μοντέλο, δεν χρησιμοποιείται από τις καθημερινές ροές |
