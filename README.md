# testmail
Kernidee: combineer mutual TLS op transportniveau (server én client authenticeren met X.509) met message‑level handtekeningen (S/MIME) en DNS‑gepubliceerde sleutels (DANE/SMIMEA). Voeg policy‑handhaving toe (MTA‑STS/DMARC) en een transparantielog (CT‑achtig) om misuitgifte van certificaten detecteerbaar te maken.

1) Doel & uitgangspunten

Afzender-identiteit aantoonbaar: de ontvanger kan cryptografisch verifiëren dat de persoon met e‑mailadres X de mail ondertekend heeft.
Transportzekerheid: end‑to‑end versleuteling waar mogelijk; in elk geval verplicht TLS met serververificatie; optioneel mutual TLS met client‑certificaat bij submission.
Interoperabel & incrementeel: werkt samen met bestaande SMTP-infrastructuur; degradeert veilig naar DKIM/DMARC waar peers niet meedoen.
Beheerbaar: uitgifte, rotatie en intrekking van certificaten moeten operationeel haalbaar zijn (ACME‑achtig, MDM/IdP‑gestuurd).


2) Hoog‑over architectuurschets
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

Belangrijkste bouwstenen

S/MIME: user‑niveau ondertekening/versleuteling (X.509 met subjectAltName=rfc822Name).
Mutual TLS bij submission: client presenteert user‑cert aan de submission‑server (port 587).
DANE/SMIMEA (met DNSSEC): publiceer publieke sleutels/bindings in DNS zodat ontvangers geen “blinde” PKI hoeven te vertrouwen.
DKIM/DMARC/MTA‑STS/TLS‑RPT: beleidslaag om spoofing/downgrade tegen te gaan en zichtbaarheid te krijgen.
Transparantielog (CT‑achtig): misuitgifte detecteren voor e‑mailcertificaten (optioneel, maar aanbevolen).


3) Componenten & standaarden (korte mapping)

Transport: SMTP SUBMISSION met STARTTLS en client-certificaten; inter‑MTA via TLS, afgedwongen via MTA‑STS of DANE TLSA.
Identiteit gebruiker: S/MIME signature (X.509) + optioneel SMIMEA in DNS (DANE voor S/MIME).
Identiteit domein: DKIM signaturen + DMARC beleid.
Sleutelpublicatie: DNSSEC + TLSA/SMIMEA records.
Cert lifecycle: ACME‑achtig enrollment (IdP/MDM), rotatie, revocatie (OCSP/CRL), transparency logging.


4) End‑to‑end flow (gedetailleerd)
4.1 Uitgifte van user‑certificaat

User/device authenticeert bij IdP (bijv. Entra ID/AAD of interne IdP).
ACME‑achtig proces vraagt een S/MIME‑cert aan bij de CA, met subjectAltName=rfc822Name=user@domein.tld.
Cert wordt uitgerold via MDM/Group Policy (Windows), of in de client opgeslagen (smartcard/TPM/Keychain).
(Optioneel) SMIMEA record in DNSSEC voor dit e‑mailadres publiceren.

4.2 Verzenden (client → submission)

Client maakt e‑mail en tekent S/MIME (optioneel ook versleutelen).
Client maakt TLS-verbinding met submission server (587) en presenteert client‑cert (mutual TLS).
Server valideert client‑cert (CA trust, CRL/OCSP, SAN=rfc822Match).
Server voegt DKIM toe en zet een header Authentication-Results met “auth=pass; mTLS-user=user@domein.tld”.

4.3 Relay (MTA → MTA)

Uitgaande MTA forceert TLS t.o.v. ontvangende MTA via MTA‑STS of DANE.
(Optioneel) client‑certs ook tussen MTA’s, maar dit is lastiger in heterogene internet‑omgeving; daarom primair server‑auth op dit segment.

4.4 Ontvangen & valideren (MTA → MUA)

Ontvangende MTA verifieert TLS policy (MTA‑STS/DANE), DKIM en DMARC.
Client/MTA verifieert S/MIME signature m.b.v.:

User‑cert chain + OCSP/CRL,
SMIMEA (DNSSEC) als trust anchor/binding,
(Optioneel) CT‑achtig log: staat dit cert in een publiek/enterprise log?


UI‑indicatoren: “Geverifieerde afzender (user@domein.tld) – certificaat geldig”.


5) Backwards‑compatibility (als peers niet meedoen)

Volledige trust: TLS ok + DKIM pass + S/MIME pass (+ DNSSEC/SMIMEA match).
Degradatie: als S/MIME ontbreekt maar DKIM/DMARC pass → toon “geverifieerd domein”, geen persoons‑identiteit.
Hard fail (beleid): DMARC p=reject, MTA‑STS mode=enforce, DANE required voor kritieke stromen.


6) Bedreigingen & mitigaties

From‑spoofing persoon → S/MIME vereist; UI toont alleen “geverifieerd persoon” bij geldige handtekening.
Domain spoofing → DKIM+DMARC + MTA‑STS/DANE.
MitM/downgrade TLS → MTA‑STS en/of DANE (met DNSSEC).
Misuitgifte certs → CT‑achtig log, OCSP/CRL, korte geldigheid.
Account compromise → cert in TPM/Smartcard opslaan, revocatie afdwingen, device posture checks.
