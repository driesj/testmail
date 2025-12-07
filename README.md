# testmail
Kernidee: combineer mutual TLS op transportniveau (server én client authenticeren met X.509) met message‑level handtekeningen (S/MIME) en DNS‑gepubliceerde sleutels (DANE/SMIMEA). Voeg policy‑handhaving toe (MTA‑STS/DMARC) en een transparantielog (CT‑achtig) om misuitgifte van certificaten detecteerbaar te maken.

Helemaal! Hieronder krijg je een **architectuurschets** voor een “HTTPS-achtig” e-mailmodel met sterke afzender‑identiteit op basis van certificaten, plus een **proof‑of‑concept (PoC)** dat je lokaal kunt uitproberen met OpenSSL en (optioneel) een klein Python‑servertje. Ik houd rekening met **backwards‑compatibility** (SMTP/IMAP blijven werken), en met wat in de praktijk haalbaar is in o.a. Microsoft 365/Exchange-omgevingen.

> **Kernidee:** combineer **mutual TLS op transportniveau** (server én client authenticeren met X.509) met **message‑level handtekeningen** (S/MIME) en **DNS‑gepubliceerde sleutels** (DANE/SMIMEA). Voeg **policy‑handhaving** toe (MTA‑STS/DMARC) en een **transparantielog** (CT‑achtig) om misuitgifte van certificaten detecteerbaar te maken.

***

## 1) Doel & uitgangspunten

*   **Afzender-identiteit aantoonbaar**: de ontvanger kan cryptografisch verifiëren dat *de persoon* met e‑mailadres X de mail ondertekend heeft.
*   **Transportzekerheid**: end‑to‑end versleuteling waar mogelijk; in elk geval verplicht TLS met serververificatie; optioneel **mutual TLS** met client‑certificaat bij submission.
*   **Interoperabel & incrementeel**: werkt samen met bestaande SMTP-infrastructuur; degradeert veilig naar DKIM/DMARC waar peers niet meedoen.
*   **Beheerbaar**: uitgifte, rotatie en intrekking van certificaten moeten operationeel haalbaar zijn (ACME‑achtig, MDM/IdP‑gestuurd).

***

## 2) Hoog‑over architectuurschets

    +-------------------+        Mutual TLS (Client Cert)        +----------------------+
    |  Mail User Agent  | -------------------------------------> |  Submission Server   |
    |  (Outlook/Client) |        (SMTP SUBMISSION: 587/STARTTLS) |  (smtp.sub.example)  |
    |   - S/MIME sign   |                                        |  - Cert chain OK     |
    |   - User cert     |          DKIM (domain level)           |  - DKIM sign         |
    +---------+---------+                                        +-----------+----------+
              |                                                              |
              |                                          TLS+DANE / MTA-STS  |
              |                                                              v
              |                                                 +------------+-----------+
              |                                                 | Receiving MTA (MX)     |
              |                                                 |  - Verify server cert  |
              |                                                 |  - Verify DKIM         |
              |                                                 |  - Verify S/MIME user  |
              |                                                 +------------+-----------+
              |                                                              |
              |                                           S/MIME verify (user identity)
              v                                                              v
    +---------+---------+                                        +-----------+----------+
    |   PKI / CA / CT   |<--- Publish SMIMEA/DANE via DNSSEC --->|     DNS (DNSSEC)     |
    | - Issue user cert |                                         | - TLSA/SMIMEA keys  |
    | - CT-like log     |                                         | - MTA-STS policy    |
    +-------------------+                                         +---------------------+

**Belangrijkste bouwstenen**

*   **S/MIME**: user‑niveau ondertekening/versleuteling (X.509 met `subjectAltName=rfc822Name`).
*   **Mutual TLS bij submission**: client presenteert user‑cert aan de submission‑server (port 587).
*   **DANE/SMIMEA** (met DNSSEC): publiceer publieke sleutels/bindings in DNS zodat ontvangers geen “blinde” PKI hoeven te vertrouwen.
*   **DKIM/DMARC/MTA‑STS/TLS‑RPT**: beleidslaag om spoofing/downgrade tegen te gaan en zichtbaarheid te krijgen.
*   **Transparantielog (CT‑achtig)**: misuitgifte detecteren voor e‑mailcertificaten (optioneel, maar aanbevolen).

***

## 3) Componenten & standaarden (korte mapping)

*   **Transport**: SMTP SUBMISSION met **STARTTLS** en **client-certificaten**; inter‑MTA via TLS, afgedwongen via **MTA‑STS** of **DANE TLSA**.
*   **Identiteit gebruiker**: **S/MIME** signature (X.509) + optioneel **SMIMEA** in DNS (DANE voor S/MIME).
*   **Identiteit domein**: **DKIM** signaturen + **DMARC** beleid.
*   **Sleutelpublicatie**: **DNSSEC** + **TLSA/SMIMEA** records.
*   **Cert lifecycle**: ACME‑achtig enrollment (IdP/MDM), rotatie, revocatie (OCSP/CRL), **transparency logging**.

***

## 4) End‑to‑end flow (gedetailleerd)

### 4.1 Uitgifte van user‑certificaat

1.  User/device authenticeert bij IdP (bijv. Entra ID/AAD of interne IdP).
2.  **ACME‑achtig** proces vraagt een **S/MIME‑cert** aan bij de CA, met `subjectAltName=rfc822Name=user@domein.tld`.
3.  Cert wordt **uitgerold** via MDM/Group Policy (Windows), of in de client opgeslagen (smartcard/TPM/Keychain).
4.  (Optioneel) **SMIMEA** record in DNSSEC voor dit e‑mailadres publiceren.

### 4.2 Verzenden (client → submission)

1.  Client maakt e‑mail en **tekent S/MIME** (optioneel ook versleutelen).
2.  Client maakt TLS-verbinding met **submission server (587)** en presenteert **client‑cert** (mutual TLS).
3.  Server valideert client‑cert (CA trust, CRL/OCSP, SAN=rfc822Match).
4.  Server voegt **DKIM** toe en zet een header `Authentication-Results` met “**auth=pass; mTLS-user=<user@domein.tld>**”.

### 4.3 Relay (MTA → MTA)

1.  Uitgaande MTA forceert TLS t.o.v. ontvangende MTA via **MTA‑STS** of **DANE**.
2.  (Optioneel) client‑certs ook tussen MTA’s, maar dit is lastiger in heterogene internet‑omgeving; daarom primair **server‑auth** op dit segment.

### 4.4 Ontvangen & valideren (MTA → MUA)

1.  Ontvangende MTA verifieert **TLS policy** (MTA‑STS/DANE), **DKIM** en **DMARC**.
2.  Client/MTA verifieert **S/MIME signature** m.b.v.:
    *   User‑cert chain + **OCSP/CRL**,
    *   **SMIMEA** (DNSSEC) als trust anchor/binding,
    *   (Optioneel) **CT‑achtig log**: staat dit cert in een publiek/enterprise log?
3.  UI‑indicatoren: “**Geverifieerde afzender** (<user@domein.tld>) – certificaat geldig”.

***

## 5) Backwards‑compatibility (als peers niet meedoen)

*   **Volledige trust**: TLS ok + DKIM pass + S/MIME pass (+ DNSSEC/SMIMEA match).
*   **Degradatie**: als S/MIME ontbreekt maar DKIM/DMARC pass → toon “geverifieerd domein”, geen persoons‑identiteit.
*   **Hard fail** (beleid): DMARC `p=reject`, MTA‑STS `mode=enforce`, DANE required voor kritieke stromen.

***

## 6) Bedreigingen & mitigaties

*   **From‑spoofing persoon** → **S/MIME** vereist; UI toont alleen “geverifieerd persoon” bij geldige handtekening.
*   **Domain spoofing** → **DKIM+DMARC** + **MTA‑STS/DANE**.
*   **MitM/downgrade TLS** → **MTA‑STS** en/of **DANE** (met DNSSEC).
*   **Misuitgifte certs** → **CT‑achtig log**, **OCSP/CRL**, korte geldigheid.
*   **Account compromise** → cert in **TPM/Smartcard** opslaan, **revocatie** afdwingen, **device posture** checks.

***

## 7) Proof‑of‑Concept (lokale demo)

Je hebt nodig: **OpenSSL** (CLI), optioneel **Python 3**. Alle stappen zijn lokaal.

### 7.1 Root CA, server‑ en user‑certs maken (OpenSSL)

```bash
# 1) Root CA (self-signed)
openssl genrsa -out ca.key 4096
openssl req -x509 -new -key ca.key -sha256 -days 3650 -subj "/CN=Local Demo Root CA" -out ca.crt

# 2) Server cert voor submission server (smtp.sub.example)
openssl genrsa -out server.key 4096
cat > server.csr.cnf <<EOF
[req]
prompt = no
distinguished_name = dn
req_extensions = v3req
[dn]
CN = smtp.sub.example
[v3req]
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names
[alt_names]
DNS.1 = smtp.sub.example
EOF
openssl req -new -key server.key -out server.csr -config server.csr.cnf
openssl x509 -req -in server.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out server.crt -days 730 -sha256 -extensions v3req -extfile server.csr.cnf

# 3) User cert met SAN=rfc822Name (S/MIME)
openssl genrsa -out user.key 4096
cat > user.csr.cnf <<EOF
[req]
prompt = no
distinguished_name = dn
req_extensions = v3req
[dn]
CN = Dries Jans
[v3req]
keyUsage = digitalSignature, keyEncipherment, nonRepudiation
extendedKeyUsage = emailProtection, clientAuth
subjectAltName = email:dries.jans@example.com
EOF
openssl req -new -key user.key -out user.csr -config user.csr.cnf
openssl x509 -req -in user.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -out user.crt -days 365 -sha256 -extensions v3req -extfile user.csr.cnf

# (Optioneel) bundel user key+cert in PKCS#12 voor import in mailclient
openssl pkcs12 -export -inkey user.key -in user.crt -certfile ca.crt -name "Dries Jans" -out user.p12
```

### 7.2 Mutual TLS demo voor submission (zonder MTA): OpenSSL s\_server/s\_client

**Server (submission) simuleren**:

```bash
# Server luistert op TCP 587 en vereist client-cert (-Verify)
openssl s_server -accept 587 -starttls smtp \
  -cert server.crt -key server.key -CAfile ca.crt -Verify 1
```

**Client (mail user agent) simuleren**:

```bash
# Client presenteert user-cert aan server
openssl s_client -starttls smtp -connect 127.0.0.1:587 \
  -cert user.crt -key user.key -CAfile ca.crt
```

Je zou “Verification: OK” moeten zien aan serverzijde wanneer de client zich met het user‑cert presenteert.

> *Zo toon je dat “submission” op 587 **client‑certs vereist** kan maken. In Postfix kan dit echt in productie met `smtpd_tls_ask_ccert = yes` en `smtpd_tls_req_ccert = yes` op de submission service, plus een policy‑map die het SAN‑emailadres matcht.*

### 7.3 S/MIME‑ondertekenen en verifiëren

**E‑mail ondertekenen**:

```bash
cat > mail.txt <<'EOF'
From: "Dries Jans" <dries.jans@example.com>
To: test@example.net
Subject: Test S/MIME signing

Dit is een testbericht dat S/MIME-ondertekend zal worden.
EOF

openssl smime -sign -in mail.txt -text -signer user.crt -inkey user.key \
  -certfile ca.crt -out mail.signed.eml -outform pem
```

**Verifiëren** (aan ontvangerskant):

```bash
openssl smime -verify -in mail.signed.eml -CAfile ca.crt -out verified.txt
# Als OK, komt de platte tekst in verified.txt en krijg je “Verification successful”
```

> *Dit bewijst dat de **persoonlijke afzender** cryptografisch verifieerbaar is, onafhankelijk van transport.*

### 7.4 (Optioneel) Mini SMTP‑submission in Python met client‑certverplichting

> *Alleen als je wil zien hoe dit in code werkt. Je hoeft dit niet te draaien voor het concept.*

```python
import ssl, smtpd, asyncore

# Waarschuwing: smtpd/asyncore is old-school; voor demo volstaat het.
# In productie liever aiosmtpd + moderne asyncio.

class EchoServer(smtpd.SMTPServer):
    def process_message(self, peer, mailfrom, rcpttos, data, **kwargs):
        print("Peer:", peer)
        print("MAIL FROM:", mailfrom)
        print("RCPT TO:", rcpttos)
        print("DATA (first 200 chars):", data[:200])
        return

context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
context.load_cert_chain(certfile="server.crt", keyfile="server.key")
context.load_verify_locations(cafile="ca.crt")
context.verify_mode = ssl.CERT_REQUIRED  # vereis client-cert

server = EchoServer(('0.0.0.0', 2525), None)
server.channel = ssl.wrap_socket(server.channel, server_side=True, ssl_version=ssl.PROTOCOL_TLS_SERVER, do_handshake_on_connect=False, server_hostname=None, suppress_ragged_eofs=True, ciphers=None, certfile=None)

# Bovenstaande wrap is niet plug&play met smtpd; reëel: gebruik aiosmtpd+TLS proxy of inetd met stunnel.
# Voor PoC blijft OpenSSL s_server/s_client (7.2) het simpelst.

asyncore.loop()
```

> *In de praktijk is het eenvoudiger om Postfix/Exim te configureren voor mTLS op de **submission‑service** en daar beleid op te hangen (SAN=rfc822Name).*

### 7.5 SMIMEA/DANE record (conceptueel voorbeeld)

Publiceer in DNSSEC (zone gesigneerd) een **SMIMEA** record dat de user‑cert bindingt aan het e‑mailadres:

    ; hash van lokale-part (left-hand-side) volgens specificatie, voorbeeld:
    _443._tcp.smtp.sub.example.  IN  TLSA 3 1 1 <SHA-256 hash van server cert>

    ; SMIMEA voor e-mail identiteit (schematisch)
    <lhs-hash>._smimecert.example.  IN  SMIMEA 3 1 1 <SHA-256 fingerprint user.crt>

> *Ontvangers kunnen hiermee je **publieke sleutel** (of fingerprint) verifiëren via DNSSEC, zonder een externe CA te hoeven vertrouwen.*

***

## 8) Migratie/implementatie in (Microsoft) omgevingen

*   **Microsoft 365/Exchange Online**
    *   **S/MIME**: inschakelen via admin (Outlook voor Windows/Mac/OWA ondersteunen S/MIME), user‑certs uitrollen via Intune/MDM of on‑prem PKI.
    *   **DKIM**: per domein aanzetten; public keys publiceren in DNS (`selector1/selector2` CNAME/TXT).
    *   **DMARC**: beleid publiceren (`p=quarantine/reject`, `rua`/`ruf` rapportage).
    *   **TLS policy**: **MTA‑STS** + **TLS‑RPT** records publiceren; DANE vereist DNSSEC en support bij de tegenpartij (naar internet toe niet altijd haalbaar, maar binnen B2B‑peers prima).
    *   **Client‑cert bij submission**: Exchange Online laat geen client‑cert auth op SMTP AUTH zien; voor mTLS‑submission is een **edge‑relay** (bijv. Postfix/NGINX‑mail proxy) voor je uitgaande stroom een praktisch pad.
*   **On‑prem (Postfix/Exim)**
    *   Submission‑service (587) met `smtpd_tls_req_ccert = yes`, policy map die `rfc822Name` in het client‑cert matcht met `MAIL FROM`.
    *   DKIM signing met OpenDKIM; DMARC verifiëren met OpenDMARC.
    *   DNSSEC voor je zone, **MTA‑STS** policy hosten (`mta-sts.example.txt`) en **TLS‑RPT** (`_smtp._tls` TXT).

***

## 9) UI/UX‑richtlijnen (heel belangrijk voor adoptie)

*   Toon **twee badges**:
    *   **Geverifieerd persoon** ✅ (S/MIME signature valid, naam + e‑mail uit cert).
    *   **Geverifieerd domein** ✅ (DKIM/DMARC ok).
*   Waarschuw duidelijk bij:
    *   Geen S/MIME → “Persoon niet geverifieerd”.
    *   DKIM/DMARC fail → “Domein onbetrouwbaar”.
    *   TLS niet afgedwongen waar beleid dit vereist → “Transportpolicy geschonden”.

***

## 10) Wat heb je hiermee opgelost?

*   **Sterke afzendergarantie op persoonsniveau** (S/MIME + mTLS bij submission).
*   **Robuuste transportzekerheid** (MTA‑STS/DANE).
*   **Beleid & zichtbaarheid** (DMARC, TLS‑RPT).
*   **Operationele haalbaarheid** (ACME‑achtige enrollment, MDM‑uitrol, korte geldigheden).

***

## 11) Volgende stappen

1.  **Kies je trust‑model**: Publieke CA voor S/MIME of interne PKI? (Voor interne communicatie is interne PKI + SMIMEA top.)
2.  **Pilot**: opzet submission‑proxy (Postfix) met **client‑cert‑verplichting** voor een testgroep.
3.  **S/MIME rollout**: user‑certs uitrollen, Outlook/Apple Mail configureren, UI‑training.
4.  **DNS‑policies**: DKIM + DMARC + MTA‑STS/TLS‑RPT en waar mogelijk **DNSSEC+DANE/SMIMEA**.
5.  **Monitoring**: DMARC aggregate reports, TLS‑RPT, (optioneel) eigen **transparency log** voor uitgifte.

***

### Wil je dat ik:

*   een **Postfix‑submission config snippet** aanlever die je mTLS‑policy (rfc822Name‑match) afdwingt?
*   dit **PoC omzet naar een klein script** (zip) dat de certs en demo‑mail voor je aanmaakt?
*   of een **Microsoft 365‑roadmap** voor S/MIME + DKIM/DMARC + MTA‑STS specifiek voor jullie domein uitwerk?

(Geef gerust mee of je op **Microsoft 365** zit of on‑prem Exchange/Postfix, en of je **DNSSEC** beschikbaar hebt. Dan maak ik het concreet voor jouw omgeving, Dries.)

