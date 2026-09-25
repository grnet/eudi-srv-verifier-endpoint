# Τοπική εκτέλεση του Verifier Endpoint (eudi-srv-verifier-endpoint)

Οδηγίες για να τρέξει ο backend του verifier τοπικά με **HTTPS**, ώστε να τον χρησιμοποιεί το τοπικό
verifier UI και ένα Android wallet (κινητό ή emulator).

## 1. Προαπαιτούμενα

- **JDK 17 ή νεότερο** εγκατεστημένο (για να ξεκινήσει το Gradle). Έλεγχος: `java -version`.
  Το project χτίζεται με **Java 25**, αλλά το Gradle το **κατεβάζει μόνο του** την πρώτη φορά
  (χρειάζεται internet στο πρώτο build).
- Το `ca.pem` υπάρχει για το **wallet**: είναι το ίδιο αρχείο με το
  `network-logic/src/debug/res/raw/local_dev_ca.cer` του wallet.

## 2. Εκκίνηση

Σε terminal **μέσα** στον φάκελο `eudi-srv-verifier-endpoint`:

```bash
cd eudi-srv-verifier-endpoint

export SERVER_SSL_ENABLED=true
export SERVER_SSL_KEY_STORE=file:dev-certs/server-keystore.p12
export SERVER_SSL_KEY_STORE_PASSWORD=changeit
export SERVER_SSL_KEY_STORE_TYPE=PKCS12
export SERVER_SSL_KEY_ALIAS=verifier
export SPRING_PROFILES_ACTIVE=develop
export VERIFIER_PUBLICURL=https://localhost:8080

./gradlew bootRun
```

Όταν δεις στο log `Netty started on port 8080 (https)`, ο backend είναι έτοιμος. Το terminal μένει «πιασμένο»
όσο τρέχει. Σταματάς με **Ctrl+C** (όχι Ctrl+Z, αλλιώς μένει πιασμένη η πόρτα 8080).

Τι κάνει κάθε γραμμή:

| Μεταβλητή | Γιατί |
|---|---|
| `SERVER_SSL_*` | Ενεργοποιεί HTTPS με το πιστοποιητικό του `dev-certs`. Χωρίς HTTPS το wallet απορρίπτει το request. |
| `SERVER_SSL_KEY_STORE=file:dev-certs/...` | Σχετική διαδρομή → δουλεύει σε κάθε υπολογιστή, **αρκεί** το `bootRun` να τρέχει από αυτόν τον φάκελο. Το `file:` μένει. |
| `SPRING_PROFILES_ACTIVE=develop` | Υποχρεωτικό: βάζει client id `x509_hash`, που δέχεται το wallet, και αποφεύγει σφάλμα εκκίνησης (`intendedUses`). |
| `VERIFIER_PUBLICURL` | Η διεύθυνση που θα δει το wallet. **Πρέπει** να ξεκινά με `https://`, γιατί το default είναι `http://` → "url must use https". |

- Οι `export` ισχύουν **μόνο στο τρέχον terminal**. Σε νέο terminal χρειάζεται να τις ξανατρέξεις.

## 3. Σύνδεση της συσκευής

**Κινητό ή emulator — προτεινόμενο:** σε **άλλο** terminal:
```bash
adb reverse tcp:8080 tcp:8080
adb reverse --list        # πρέπει να δείχνει tcp:8080
```
Ξανατρέχει κάθε φορά που αποσυνδέεις/ξανασυνδέεις τη συσκευή ή επανεκκινείς τον emulator.

## 4. Αποδοχή του πιστοποιητικού (μία φορά)

- **Στον υπολογιστή:** άνοιξε `https://localhost:8080` στον browser που θα χρησιμοποιήσεις για το UI →
  **Advanced → Proceed**.
- **Στη συσκευή:** άνοιξε `https://localhost:8080` (ή `https://10.0.2.2:8080`) στον browser της συσκευής → αποδοχή.

## 5. Έλεγχος ότι δουλεύει

```bash
curl -k https://localhost:8080/actuator/health 2>/dev/null || curl -k -I https://localhost:8080
```
Οποιαδήποτε απάντηση HTTP (ακόμα και 404) σημαίνει ότι ο server ακούει με HTTPS.

Μετά ξεκινάς το verifier UI (δες το `InstructionsToRunServerUI.md` του `grnet/eudi-web-verifier`, στο branch demoVerifierUI4v10).