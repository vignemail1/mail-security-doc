---
title: "Troubleshoot"
---
# 10. Dépannage et Bonnes Pratiques

Ce chapitre répertorie les problèmes courants rencontrés lors de la mise en œuvre de SPF, DKIM, DMARC et ARC, ainsi que les bonnes pratiques pour maintenir une configuration optimale.

## 10.1 Problèmes courants SPF

### Problème : SPF fail malgré une configuration correcte

**Symptômes :**

- Emails rejetés ou marqués comme spam
- Logs montrant `SPF fail` ou `SPF softfail`

**Causes possibles :**

#### 1. IP source non incluse dans l'enregistrement SPF

```bash
# Diagnostic
grep "SPF fail" /var/log/mail.log | tail -10
# Identifier l'IP source dans les logs

# Vérifier l'enregistrement SPF actuel
dig TXT exemple.fr +short | grep spf
```

**Solution :**

```dns
# Avant
v=spf1 mx -all

# Après (ajouter l'IP manquante)
v=spf1 mx ip4:192.0.2.10 ip4:198.51.100.20 -all
```

#### 2. Dépassement de la limite de lookups DNS (10 maximum)

```bash
# Tester le nombre de lookups
curl -s "https://www.kitterman.com/spf/validate.html?domain=exemple.fr" | grep -i lookup
```

**Solution :**

```dns
# Problème (trop de includes)
v=spf1 include:_spf.google.com include:spf.protection.outlook.com include:sendgrid.net include:mailgun.org include:servers.mcsv.net -all

# Solution : Utiliser ip4/ip6 plutôt que include quand possible
v=spf1 ip4:192.0.2.10 include:_spf.google.com include:sendgrid.net -all

# Ou créer des enregistrements intermédiaires
_spf1.exemple.fr IN TXT "v=spf1 include:_spf.google.com include:spf.protection.outlook.com -all"
_spf2.exemple.fr IN TXT "v=spf1 include:sendgrid.net include:mailgun.org -all"

exemple.fr IN TXT "v=spf1 include:_spf1.exemple.fr include:_spf2.exemple.fr -all"
```

#### 3. Forwarding d'emails cassant SPF

**Problème :** Les emails forwardés échouent SPF car l'IP source change.

**Solution :**

- Implémenter SRS (Sender Rewriting Scheme)
- Configurer ARC pour préserver l'authentification

```bash
# Installer PostSRSd (Debian/Ubuntu)
apt install -y postsrsd

# Configuration /etc/postsrsd/postsrsd.conf
SRS_DOMAIN=exemple.fr
SRS_SECRET=/etc/postsrsd/secret
SRS_FORWARD_PORT=10001
SRS_REVERSE_PORT=10002

# Intégrer à Postfix (main.cf)
sender_canonical_maps = tcp:localhost:10001
sender_canonical_classes = envelope_sender
recipient_canonical_maps = tcp:localhost:10002
recipient_canonical_classes = envelope_recipient

# Redémarrer
systemctl restart postsrsd postfix
```

### Problème : SPF PermError

**Causes :**

- Syntaxe invalide
- Enregistrement trop long (> 255 caractères par champ TXT)
- Enregistrements SPF multiples

**Diagnostic :**

```bash
# Vérifier la syntaxe
dig TXT exemple.fr +short

# Compter les enregistrements SPF
dig TXT exemple.fr +short | grep -c "v=spf1"
# Résultat doit être 1
```

**Solution :**

```dns
# INCORRECT (deux enregistrements SPF)
exemple.fr. IN TXT "v=spf1 mx -all"
exemple.fr. IN TXT "v=spf1 ip4:192.0.2.10 -all"

# CORRECT (un seul enregistrement)
exemple.fr. IN TXT "v=spf1 mx ip4:192.0.2.10 -all"
```

Pour enregistrement trop long :

```dns
# Diviser en plusieurs chaînes
exemple.fr. IN TXT (
    "v=spf1 mx ip4:192.0.2.10 ip4:192.0.2.11 "
    "ip4:192.0.2.12 include:_spf.google.com -all"
)
```

## 10.2 Problèmes courants DKIM

### Problème : Emails non signés DKIM

**Symptômes :**

- En-tête `DKIM-Signature` absente
- `DKIM: none` dans Authentication-Results

**Diagnostic :**

```bash
# Vérifier le statut OpenDKIM
systemctl status opendkim

# Vérifier les logs
journalctl -u opendkim --since "1 hour ago" | grep -i error

# Tester la connexion milter
telnet localhost 8891
# Attendu: Connexion réussie

# Vérifier la configuration Postfix
postconf | grep milter
```

**Causes et solutions :**

#### 1. OpenDKIM non actif ou plantage

```bash
# Redémarrer OpenDKIM
systemctl restart opendkim

# Si échec de démarrage, vérifier config
opendkim -n /etc/opendkim.conf
```

#### 2. Permissions incorrectes sur les clés

```bash
# Vérifier permissions
ls -la /etc/opendkim/keys/exemple.fr/

# Correction
chown -R opendkim:opendkim /etc/opendkim
chmod 700 /etc/opendkim/keys
chmod 600 /etc/opendkim/keys/exemple.fr/*.private.key

# Redémarrer
systemctl restart opendkim
```

#### 3. Configuration milter incorrecte

```bash
# Vérifier main.cf
grep -E "milter|8891" /etc/postfix/main.cf

# Configuration correcte attendue:
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:127.0.0.1:8891
non_smtpd_milters = $smtpd_milters
```

#### 4. Tables DKIM incorrectes

**Vérifier `/etc/opendkim/KeyTable` :**

```
# Format: name domain:selector:keyfile
default._domainkey.exemple.fr exemple.fr:default:/etc/opendkim/keys/exemple.fr/default.private.key
```

**Vérifier `/etc/opendkim/SigningTable` :**

```
# Tout domaine exemple.fr utilise la clé default
*@exemple.fr default._domainkey.exemple.fr

# Ou spécifique par utilisateur
user@exemple.fr default._domainkey.exemple.fr
*@subdomain.exemple.fr subdomain._domainkey.exemple.fr
```

### Problème : DKIM fail (signature invalide)

**Symptômes :**

- En-tête DKIM-Signature présente mais `DKIM: fail`

**Causes possibles :**

#### 1. Clé publique DNS incorrecte ou absente

```bash
# Vérifier la clé publique
dig TXT default._domainkey.exemple.fr +short

# Comparer avec la clé privée
opendkim-testkey -d exemple.fr -s default -vvv

# Résultat attendu:
opendkim-testkey: key OK
```

**Solution :** Republier la clé publique correcte

```bash
# Extraire la clé publique
cat /etc/opendkim/keys/exemple.fr/default.public.txt

# Publier dans DNS (retirer espaces et quotes)
```

#### 2. Modification du message après signature

**Causes :**

- Liste de diffusion ajoutant footer
- Antivirus modifiant les pièces jointes
- Gateway ajoutant disclaimers

**Solution :**

```conf
# /etc/opendkim.conf
# Utiliser canonicalisation relaxed
Canonicalization relaxed/relaxed

# Signer uniquement les en-têtes stables
SignHeaders From,To,Subject,Date,Message-ID

# Oversigner pour éviter ajout d'en-têtes
OversignHeaders From
```

#### 3. Expiration de la signature

```bash
# Vérifier l'en-tête DKIM-Signature
# Paramètre x= indique timestamp d'expiration

# Configuration /etc/opendkim.conf
# Signature valide 7 jours
SignatureAlgorithm rsa-sha256
SignatureTTL 604800
```

### Problème : DKIM signature trop faible

**Symptômes :**

- Warnings sur clé < 2048 bits
- Certains fournisseurs rejettent

**Solution : Regénérer avec clé 2048 bits**

```bash
# Générer nouvelle clé 2048 bits
cd /etc/opendkim/keys/exemple.fr
opendkim-genkey -b 2048 -d exemple.fr -s default202602

# Publier la nouvelle clé DNS
cat default202602.txt

# Une fois propagé, mettre à jour KeyTable et SigningTable
vim /etc/opendkim/KeyTable
vim /etc/opendkim/SigningTable

# Redémarrer
systemctl restart opendkim

# Après 7 jours, supprimer ancienne clé
```

## 10.3 Problèmes courants DMARC

### Problème : DMARC fail malgré SPF et DKIM pass

**Cause : Problème d'alignement**

**Diagnostic :**

```bash
# Examiner les en-têtes d'un email rejeté
# Comparer:
From: user@exemple.fr              # Header From
Return-Path: <bounce@autre.com>    # Envelope From (SPF)
DKIM d=autre.com                    # Domaine DKIM
```

**Explication :**

- **SPF alignment** : `From` domain doit correspondre à `Return-Path` domain
- **DKIM alignment** : `From` domain doit correspondre à `d=` dans DKIM-Signature

**Solutions :**

#### Solution 1 : Utiliser alignement relaxed

```dns
# _dmarc.exemple.fr
v=DMARC1; p=quarantine; aspf=r; adkim=r; rua=mailto:dmarc@exemple.fr

# aspf=r : alignement SPF relaxed (sous-domaine OK)
# adkim=r : alignement DKIM relaxed (sous-domaine OK)
```

#### Solution 2 : Corriger la configuration d'envoi

```bash
# Assurer Return-Path = domaine From:
# /etc/postfix/main.cf
myorigin = $mydomain

# Si utilisation VERP ou SRS
sender_canonical_maps = regexp:/etc/postfix/sender_canonical
```

```
# /etc/postfix/sender_canonical
/@autre\.com$/  bounce@exemple.fr
```

#### Solution 3 : Signer DKIM avec bon domaine

```bash
# /etc/opendkim/SigningTable
# S'assurer que d= correspond au From: domain
*@exemple.fr default._domainkey.exemple.fr
```

### Problème : Pas de rapports DMARC reçus

**Causes possibles :**

#### 1. Adresse email invalide ou non monitored

```bash
# Vérifier l'enregistrement
dig TXT _dmarc.exemple.fr +short

# Vérifier que l'adresse reçoit bien les emails
echo "Test" | mail -s "Test" dmarc@exemple.fr
```

#### 2. Format URI incorrect

```dns
# INCORRECT
v=DMARC1; p=none; rua=dmarc@exemple.fr

# CORRECT
v=DMARC1; p=none; rua=mailto:dmarc@exemple.fr
```

#### 3. Validation de domaine externe

Si `rua=mailto:dmarc@autre-domaine.com`, le domaine externe doit autoriser :

```dns
# autre-domaine.com doit publier:
exemple.fr._report._dmarc.autre-domaine.com. IN TXT "v=DMARC1"
```

#### 4. Volume d'emails trop faible

Les fournisseurs n'envoient souvent des rapports que si volume significatif (> 100 emails/jour).

### Problème : Rapports DMARC illisibles

**Solution : Parser les rapports XML**

```bash
# Installer parseur DMARC
pip3 install dmarc-metrics-exporter

# Ou utiliser script Python
cat << 'EOF' > parse-dmarc.py
#!/usr/bin/env python3
import gzip
import xml.etree.ElementTree as ET
import sys
from email import message_from_file
from io import BytesIO

def parse_dmarc_report(file_path):
    with open(file_path, 'rb') as f:
        msg = message_from_file(f)

    for part in msg.walk():
        if part.get_content_type() in ['application/gzip', 'application/x-gzip']:
            payload = part.get_payload(decode=True)
            xml_data = gzip.decompress(payload)
            root = ET.fromstring(xml_data)

            # Metadata
            org = root.find('.//org_name').text
            date_begin = root.find('.//date_range/begin').text
            date_end = root.find('.//date_range/end').text

            print(f"Rapport de: {org}")
            print(f"Période: {date_begin} - {date_end}")
            print()

            # Records
            for record in root.findall('.//record'):
                source_ip = record.find('.//source_ip').text
                count = record.find('.//count').text

                spf = record.find('.//policy_evaluated/spf').text
                dkim = record.find('.//policy_evaluated/dkim').text
                dmarc = record.find('.//policy_evaluated/disposition').text

                print(f"IP: {source_ip}, Count: {count}")
                print(f"  SPF: {spf}, DKIM: {dkim}, DMARC: {dmarc}")
                print()

if __name__ == "__main__":
    if len(sys.argv) < 2:
        print(f"Usage: {sys.argv[0]} <dmarc_report.eml>")
        sys.exit(1)

    parse_dmarc_report(sys.argv[1])
EOF

chmod +x parse-dmarc.py
```

## 10.4 Problèmes courants ARC

### Problème : ARC seal invalide

**Symptômes :**

- `ARC: fail` dans Authentication-Results

**Diagnostic :**

```bash
# Vérifier OpenARC
systemctl status openarc
journalctl -u openarc | tail -50

# Vérifier clé publique DNS
dig TXT arc._domainkey.exemple.fr +short
```

**Causes et solutions :**

#### 1. Chaîne ARC cassée

ARC nécessite que TOUS les intermédiaires signent. Si un maillon manque, la chaîne est invalide.

**Solution :** S'assurer que tous les serveurs de forwarding supportent ARC.

#### 2. Configuration incorrecte

```conf
# /etc/openarc/openarc.conf

# Mode doit être 'sv' (sign and verify)
Mode sv

# AuthservID doit correspondre au serveur
AuthservID mail.exemple.fr

# TrustedAuthservIDs
TrustedAuthservIDs mail.exemple.fr,autre-serveur.com
```

## 10.5 Problèmes de délivrabilité

### Emails arrivés en spam malgré SPF/DKIM/DMARC pass

**Autres facteurs impactant la délivrabilité :**

#### 1. Réputation IP/Domaine

```bash
# Vérifier blacklists
for bl in zen.spamhaus.org b.barracudacentral.org bl.spamcop.net; do
    echo -n "$bl: "
    if host 10.2.0.192.$bl > /dev/null 2>&1; then
        echo "BLACKLISTED"
    else
        echo "OK"
    fi
done

# Vérifier réputation Senderscore
curl "https://www.senderscore.org/lookup.php?lookup=192.0.2.10"
```

**Solution :**

- Demander delisting si nécessaire
- Améliorer pratiques d'envoi (opt-in, bounce handling)
- Utiliser FBL (Feedback Loops)

#### 2. Contenu du message

**Facteurs pénalisants :**

- Mots-clés spam ("gratuit", "cliquez ici", all caps)
- Ratio texte/images déséquilibré
- URLs raccourcies
- Pièces jointes exécutables

**Solution : Tester avec SpamAssassin**

```bash
# Installer SpamAssassin
apt install -y spamassassin

# Tester un email
spamassassin -t < email.txt

# Score < 5 = OK
# Score > 5 = Risque spam
```

#### 3. Configuration serveur

```bash
# Vérifier PTR (reverse DNS)
dig -x 192.0.2.10 +short
# Doit retourner: mail.exemple.fr

# Vérifier HELO/EHLO
telnet mail.exemple.fr 25
EHLO test
# Doit correspondre au PTR

# Vérifier certificat SSL
openssl s_client -connect mail.exemple.fr:587 -starttls smtp
```

### Emails rejetés immédiatement

**Vérifier les logs Postfix :**

```bash
grep "reject" /var/log/mail.log | tail -20
```

**Raisons courantes :**

#### 1. Greylisting

```
feb 10 12:00:00 mail postfix/smtpd[1234]: NOQUEUE: reject: 450 4.7.1 Greylisting in effect
```

**Solution :** Réessayer automatiquement (comportement normal).

#### 2. Rate limiting

```
feb 10 12:00:00 mail postfix/smtpd[1234]: reject: 450 4.7.1 Too many connections
```

**Solution :** Espacer les envois, augmenter les limites si légitime.

```conf
# /etc/postfix/main.cf
smtpd_client_connection_rate_limit = 100
smtpd_client_message_rate_limit = 100
```

#### 3. Politique stricte du destinataire

```
feb 10 12:00:00 mail postfix/smtp[1234]: 550 5.7.1 DMARC policy reject
```

**Solution :** S'assurer DMARC pass avec alignement strict.

## 10.6 Bonnes pratiques générales

### SPF

#### ✅ Faire

- Utiliser `-all` en production (après tests)
- Inclure toutes les sources d'envoi légitimes
- Rester sous la limite de 10 lookups DNS
- Tester avec plusieurs outils avant déploiement
- Documenter chaque mécanisme

```dns
# Exemple bien documenté
v=spf1
  mx                          # Serveurs MX du domaine
  ip4:192.0.2.10              # Serveur mail principal
  ip4:198.51.100.0/24         # Subnet serveurs backup
  include:_spf.google.com     # Google Workspace
  -all                        # Rejeter tout le reste
```

#### ❌ Éviter

- Utiliser `+all` (autorise tout, inutile)
- Trop de mécanismes `include:` (limite de lookups)
- Oublier des sources légitimes (services tiers)
- Enregistrements SPF multiples

### DKIM

#### ✅ Faire

- Utiliser clés 2048 bits minimum
- Rotation des clés tous les 6-12 mois
- Signer les en-têtes importants : From, To, Subject, Date
- Utiliser `Canonicalization relaxed/relaxed`
- Tester la signature après chaque changement

```bash
# Script de rotation planifiée
#!/bin/bash
# /usr/local/bin/rotate-dkim.sh

DOMAIN="exemple.fr"
NEW_SELECTOR="sel$(date +%Y%m)"

opendkim-genkey -b 2048 -d $DOMAIN -s $NEW_SELECTOR
echo "Nouvelle clé générée: $NEW_SELECTOR"
echo "Publiez dans DNS puis mettez à jour la config OpenDKIM"
```

#### ❌ Éviter

- Clés < 1024 bits (dépréciées)
- Permissions incorrectes sur clés privées
- Signer trop d'en-têtes (risque de modification)
- Oublier de tester après rotation de clé

### DMARC

#### ✅ Faire

- **Phase 1** (1-2 mois) : `p=none` avec rapports (monitoring)
- **Phase 2** (1-2 mois) : `p=quarantine` avec `pct=10` puis augmentation progressive
- **Phase 3** : `p=reject` une fois stabilisé
- Analyser les rapports régulièrement
- Configurer `rua=` ET `ruf=` pour rapports détaillés

```dns
# Progression recommandée

# Étape 1: Monitoring (1 mois)
v=DMARC1; p=none; rua=mailto:dmarc-reports@exemple.fr; ruf=mailto:dmarc-forensic@exemple.fr; pct=100; fo=1

# Étape 2: Quarantine progressif (1 mois)
v=DMARC1; p=quarantine; rua=mailto:dmarc-reports@exemple.fr; pct=10
# Après 1 semaine sans problème : pct=50
# Après 2 semaines : pct=100

# Étape 3: Enforcement strict
v=DMARC1; p=reject; rua=mailto:dmarc-reports@exemple.fr; pct=100
```

#### ❌ Éviter

- Passer directement à `p=reject` sans monitoring
- Ignorer les rapports reçus
- Ne pas configurer d'adresse `rua=`
- Utiliser `p=none` indéfiniment en production

### ARC

#### ✅ Faire

- Implémenter si vous opérez :
  - Listes de diffusion
  - Services de forwarding
  - Gateways email
- Vérifier que tous les serveurs de la chaîne supportent ARC
- Utiliser les mêmes bonnes pratiques que DKIM pour les clés

#### ❌ Éviter

- Implémenter ARC sur serveur final (inutile)
- Casser la chaîne ARC (ne pas signer un forward)

### Sécurité des clés

```bash
# Permissions correctes
chown -R opendkim:opendkim /etc/opendkim/keys
chmod 750 /etc/opendkim/keys
chmod 750 /etc/opendkim/keys/exemple.fr
chmod 640 /etc/opendkim/keys/exemple.fr/*.private.key

# Backup chiffré des clés
tar czf - /etc/opendkim/keys | \
    gpg --encrypt --recipient admin@exemple.fr > \
    /backup/opendkim-keys-$(date +%Y%m%d).tar.gz.gpg

# Stocker backup hors serveur
rsync /backup/opendkim-keys-*.gpg backup-server:/secure/
```

## 10.7 Monitoring et alertes

### Scripts de monitoring

#### Vérification quotidienne des services

```bash
#!/bin/bash
# /usr/local/bin/check-email-auth-health.sh

ALERT_EMAIL="admin@exemple.fr"
ERRORS=""

# Vérifier services
for service in postfix opendkim opendmarc; do
    if ! systemctl is-active --quiet $service; then
        ERRORS="${ERRORS}Service $service est arrêté\n"
    fi
done

# Vérifier DNS
if ! dig +short TXT exemple.fr | grep -q spf; then
    ERRORS="${ERRORS}Enregistrement SPF absent ou inaccessible\n"
fi

if ! dig +short TXT default._domainkey.exemple.fr | grep -q "v=DKIM1"; then
    ERRORS="${ERRORS}Clé DKIM absente ou inaccessible\n"
fi

if ! dig +short TXT _dmarc.exemple.fr | grep -q "v=DMARC1"; then
    ERRORS="${ERRORS}Enregistrement DMARC absent ou inaccessible\n"
fi

# Vérifier erreurs récentes dans logs
RECENT_ERRORS=$(journalctl --since "1 hour ago" | grep -i "opendkim\|opendmarc" | grep -i "error\|fail" | wc -l)

if [ $RECENT_ERRORS -gt 10 ]; then
    ERRORS="${ERRORS}$RECENT_ERRORS erreurs DKIM/DMARC dans la dernière heure\n"
fi

# Envoyer alerte si problèmes
if [ -n "$ERRORS" ]; then
    echo -e "$ERRORS" | mail -s "[ALERTE] Problèmes authentification email" "$ALERT_EMAIL"
    exit 1
fi

exit 0
```

```bash
chmod +x /usr/local/bin/check-email-auth-health.sh

# Cron toutes les heures
echo "0 * * * * /usr/local/bin/check-email-auth-health.sh" | crontab -
```

#### Dashboard de statistiques

```bash
#!/bin/bash
# /usr/local/bin/email-auth-stats.sh

echo "=== Statistiques Authentification Email ==="
echo "Date: $(date)"
echo ""

# Statistiques SPF (dernières 24h)
echo "--- SPF (24h) ---"
SPF_PASS=$(grep -c "SPF pass" /var/log/mail.log | tail -1000)
SPF_FAIL=$(grep -c "SPF fail" /var/log/mail.log | tail -1000)
echo "Pass: $SPF_PASS"
echo "Fail: $SPF_FAIL"

# Statistiques DKIM
echo ""
echo "--- DKIM (24h) ---"
DKIM_SIGN=$(grep -c "DKIM-Signature" /var/log/mail.log | tail -1000)
DKIM_PASS=$(grep -c "dkim=pass" /var/log/mail.log | tail -1000)
echo "Signés: $DKIM_SIGN"
echo "Vérifiés pass: $DKIM_PASS"

# Statistiques DMARC
echo ""
echo "--- DMARC (24h) ---"
DMARC_PASS=$(grep -c "dmarc=pass" /var/log/mail.log | tail -1000)
DMARC_FAIL=$(grep -c "dmarc=fail" /var/log/mail.log | tail -1000)
echo "Pass: $DMARC_PASS"
echo "Fail: $DMARC_FAIL"

# Top IPs rejetées
echo ""
echo "--- Top 5 IPs rejetées SPF ---"
grep "SPF fail" /var/log/mail.log | \
    grep -oP 'client=\S+\[\K[0-9.]+' | \
    sort | uniq -c | sort -rn | head -5

echo ""
echo "==================================="
```

### Intégration avec outils de monitoring

#### Prometheus + Grafana

```bash
# Installer node_exporter avec textfile collector
wget https://github.com/prometheus/node_exporter/releases/download/v1.7.0/node_exporter-1.7.0.linux-amd64.tar.gz
tar xzf node_exporter-1.7.0.linux-amd64.tar.gz
cp node_exporter-1.7.0.linux-amd64/node_exporter /usr/local/bin/

# Créer script collecteur
cat << 'EOF' > /usr/local/bin/email-auth-metrics.sh
#!/bin/bash

OUTPUT="/var/lib/node_exporter/textfile_collector/email_auth.prom"
mkdir -p $(dirname $OUTPUT)

SPF_PASS=$(grep -c "SPF pass" /var/log/mail.log | tail -1000 || echo 0)
SPF_FAIL=$(grep -c "SPF fail" /var/log/mail.log | tail -1000 || echo 0)
DKIM_PASS=$(grep -c "dkim=pass" /var/log/mail.log | tail -1000 || echo 0)
DKIM_FAIL=$(grep -c "dkim=fail" /var/log/mail.log | tail -1000 || echo 0)
DMARC_PASS=$(grep -c "dmarc=pass" /var/log/mail.log | tail -1000 || echo 0)
DMARC_FAIL=$(grep -c "dmarc=fail" /var/log/mail.log | tail -1000 || echo 0)

cat > $OUTPUT << METRICS
# HELP spf_pass_total Total SPF pass
# TYPE spf_pass_total counter
spf_pass_total $SPF_PASS

# HELP spf_fail_total Total SPF fail
# TYPE spf_fail_total counter
spf_fail_total $SPF_FAIL

# HELP dkim_pass_total Total DKIM pass
# TYPE dkim_pass_total counter
dkim_pass_total $DKIM_PASS

# HELP dkim_fail_total Total DKIM fail
# TYPE dkim_fail_total counter
dkim_fail_total $DKIM_FAIL

# HELP dmarc_pass_total Total DMARC pass
# TYPE dmarc_pass_total counter
dmarc_pass_total $DMARC_PASS

# HELP dmarc_fail_total Total DMARC fail
# TYPE dmarc_fail_total counter
dmarc_fail_total $DMARC_FAIL
METRICS
EOF

chmod +x /usr/local/bin/email-auth-metrics.sh

# Cron toutes les 5 minutes
echo "*/5 * * * * /usr/local/bin/email-auth-metrics.sh" | crontab -
```

## 10.8 Checklist de maintenance

### Quotidienne

```markdown
☐ Vérifier statut services (postfix, opendkim, opendmarc)
☐ Surveiller logs pour erreurs critiques
☐ Vérifier queue emails (mailq)
```

### Hebdomadaire

```markdown
☐ Analyser rapports DMARC reçus
☐ Vérifier statistiques SPF/DKIM/DMARC
☐ Tester envoi emails vers principaux fournisseurs
☐ Vérifier blacklists
```

### Mensuelle

```markdown
☐ Vérifier expiration certificats SSL
☐ Analyser tendances délivrabilité
☐ Audit configuration (postconf -n)
☐ Backup des clés DKIM
☐ Vérifier propagation DNS (SPF, DKIM, DMARC)
```

### Semestrielle

```markdown
☐ Rotation clés DKIM
☐ Audit sécurité complet
☐ Mise à jour système et packages
☐ Revue politique DMARC (envisager renforcement)
☐ Test de restauration depuis backup
```

### Annuelle

```markdown
☐ Revue architecture complète
☐ Mise à jour documentation
☐ Formation équipe
☐ Audit externe si requis
```

## 10.9 Documentation et communication

### Documenter les changements

```bash
# Template changelog
cat << 'EOF' > /etc/email-auth/CHANGELOG.md
# Changelog - Authentification Email

## 2026-02-10
### Ajouté
- Nouveau sélecteur DKIM "default202602" (clé 2048 bits)
- Script de monitoring Prometheus

### Modifié
- Politique DMARC : p=quarantine → p=reject
- OpenDKIM : Canonicalization simple/simple → relaxed/relaxed

### Supprimé
- Ancien sélecteur DKIM "default202501"

### Sécurité
- Rotation clés DKIM (planifiée 2026-08-10)

## 2026-01-15
### Ajouté
- Configuration ARC pour listes de diffusion
...
EOF
```

### Communiquer les changements

**Avant un changement majeur (ex: DMARC p=reject) :**

```markdown
À: Équipe IT, Stakeholders
Sujet: [IMPORTANT] Activation DMARC strict le 2026-03-01

Bonjour,

Nous allons renforcer notre politique DMARC le 1er mars 2026.

**Changement:**
Passage de p=quarantine à p=reject pour exemple.fr

**Impact:**

- Emails non authentifiés seront rejetés (non délivrés)
- Meilleure protection contre phishing/spoofing
- Réduction du spam usurpant notre domaine

**Actions requises:**

1. Vérifier que tous les services d'envoi sont configurés (SPF/DKIM)
2. Tester envois depuis applications internes
3. Signaler tout problème avant le 25 février

**Monitoring:**
Suivi quotidien pendant 2 semaines après activation.

Questions: admin@exemple.fr
```

## 10.10 Résumé des commandes utiles

```bash
### Diagnostic rapide
# Vérifier DNS
dig TXT exemple.fr +short | grep spf
dig TXT default._domainkey.exemple.fr +short
dig TXT _dmarc.exemple.fr +short

# Vérifier services
systemctl status postfix opendkim opendmarc

# Logs en temps réel
tail -f /var/log/mail.log | grep -E "SPF|DKIM|DMARC"

# Tester envoi
echo "Test" | mail -s "Test" check-auth@verifier.port25.com

### Dépannage
# Redémarrer services
systemctl restart opendkim opendmarc postfix

# Vérifier configuration Postfix
postconf -n | grep -E "milter|spf"
postfix check

# Tester clé DKIM
opendkim-testkey -d exemple.fr -s default -vvv

# Vider queue Postfix
postqueue -f

# Analyser queue
postqueue -p

### Statistiques
# SPF/DKIM/DMARC pass/fail
grep -E "SPF|DKIM|DMARC" /var/log/mail.log | \
    grep -oE "(SPF|DKIM|DMARC)\s+(pass|fail)" | \
    sort | uniq -c

# Top domaines défaillants
grep "DMARC fail" /var/log/mail.log | \
    grep -oP 'from=<.*@\K[^>]+' | \
    sort | uniq -c | sort -rn | head -10
```

## Conclusion

Le dépannage efficace des mécanismes d'authentification email repose sur :

1. **Compréhension approfondie** : Connaître le fonctionnement de SPF, DKIM, DMARC et ARC
2. **Monitoring proactif** : Détecter les problèmes avant qu'ils n'impactent la délivrabilité
3. **Documentation rigoureuse** : Tracer tous les changements et configurations
4. **Tests réguliers** : Valider continuellement le bon fonctionnement
5. **Réactivité** : Corriger rapidement les défaillances détectées

En suivant les bonnes pratiques et checklist de ce chapitre, vous maintiendrez une infrastructure email sécurisée et fiable.
