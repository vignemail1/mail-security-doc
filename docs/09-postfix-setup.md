---
title: "Mise en place"
---
# Guide de Mise en Place avec Postfix sur Linux

Ce chapitre détaille l'implémentation complète de SPF, DKIM, DMARC et ARC sur un serveur Linux utilisant Postfix comme MTA (Mail Transfer Agent).

## Prérequis

### Environnement

- **OS** : Debian 11/12, Ubuntu 20.04/22.04/24.04, Rocky Linux 8/9, ou AlmaLinux
- **Postfix** : Version 3.5 ou supérieure
- **DNS** : Accès administrateur aux enregistrements DNS du domaine
- **Domaine** : Un nom de domaine valide et fonctionnel

### Informations nécessaires

```bash
# Variables d'environnement pour ce guide
export DOMAIN="exemple.fr"
export HOSTNAME="mail.exemple.fr"
export IP_PUBLIQUE="192.0.2.10"
export SELECTOR="default"
```

### Architecture cible

```mermaid
flowchart TB
    subgraph MAIL["Serveur Mail Linux<br/>mail.exemple.fr — 192.0.2.10"]
        direction TB

        POSTFIX["Postfix MTA<br/><br/>• SMTP entrant / sortant"]

        subgraph FILTERS["Services de filtrage / authentification"]
            direction LR
            OPENDKIM["OpenDKIM<br/>Port 8891"]
            OPENDMARC["OpenDMARC<br/>Port 8893"]
            SPF["policyd-spf-python"]
        end

        OPENARC["OpenARC — optionnel<br/>Port 8892"]

        POSTFIX --> OPENDKIM
        POSTFIX --> OPENDMARC
        POSTFIX --> SPF
        POSTFIX -.-> OPENARC

        OPENDKIM --> SPF
        OPENDMARC --> SPF
    end

    subgraph DNS_SERVER["Serveur DNS — exemple.fr"]
        direction TB
        SPF_DNS["SPF<br/>TXT exemple.fr"]
        DKIM_DNS["DKIM<br/>TXT default._domainkey.exemple.fr"]
        DMARC_DNS["DMARC<br/>TXT _dmarc.exemple.fr"]
        PTR_DNS["PTR<br/>192.0.2.10 → mail.exemple.fr"]
    end

    MAIL -->|Requêtes DNS| DNS_SERVER
```

## Installation de Postfix

### Debian/Ubuntu

```bash
# Mise à jour du système
apt update && apt upgrade -y

# Installation de Postfix
apt install -y postfix postfix-policyd-spf-python

# Pendant l'installation, choisir:
# - Type: Internet Site
# - System mail name: exemple.fr
```

### Rocky Linux / AlmaLinux

```bash
# Mise à jour
dnf update -y

# Installation EPEL
dnf install -y epel-release

# Installation Postfix
dnf install -y postfix pypolicyd-spf

# Activer et démarrer Postfix
systemctl enable postfix
systemctl start postfix
```

### Configuration de base de Postfix

Éditer `/etc/postfix/main.cf` :

```bash
# Sauvegarder la config originale
cp /etc/postfix/main.cf /etc/postfix/main.cf.orig

# Éditer la configuration
vim /etc/postfix/main.cf
```

```conf
# /etc/postfix/main.cf

# Informations de base
myhostname = mail.exemple.fr
mydomain = exemple.fr
myorigin = $mydomain

# Destinations
inet_interfaces = all
inet_protocols = ipv4
mydestination = $myhostname, localhost.$mydomain, localhost, $mydomain

# Relayage
relayhost =
mynetworks = 127.0.0.0/8, 192.168.1.0/24

# Limites de taille
message_size_limit = 52428800
mailbox_size_limit = 0

# Banner SMTP
smtpd_banner = $myhostname ESMTP

# TLS pour SMTP sortant
smtp_tls_security_level = may
smtp_tls_loglevel = 1

# TLS pour SMTP entrant
smtpd_tls_security_level = may
smtpd_tls_cert_file = /etc/ssl/certs/mail.exemple.fr.crt
smtpd_tls_key_file = /etc/ssl/private/mail.exemple.fr.key
smtpd_tls_loglevel = 1
smtpd_tls_received_header = yes

# Restrictions SMTP
smtpd_helo_required = yes
smtpd_helo_restrictions =
    permit_mynetworks,
    reject_invalid_helo_hostname,
    reject_non_fqdn_helo_hostname

smtpd_sender_restrictions =
    permit_mynetworks,
    reject_non_fqdn_sender,
    reject_unknown_sender_domain

smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    reject_invalid_hostname,
    reject_non_fqdn_recipient,
    reject_unknown_recipient_domain
```

## Configuration DNS préalable

### Reverse DNS (PTR)

Contacter votre fournisseur d'hébergement pour configurer :

```dns
10.2.0.192.in-addr.arpa. IN PTR mail.exemple.fr.
```

Vérification :

```bash
dig -x 192.0.2.10 +short
# Résultat attendu: mail.exemple.fr.
```

### Enregistrement A

```dns
mail.exemple.fr. IN A 192.0.2.10
```

### Enregistrement MX

```dns
exemple.fr. IN MX 10 mail.exemple.fr.
```

Vérification :

```bash
dig MX exemple.fr +short
# Résultat attendu: 10 mail.exemple.fr.
```

## Configuration SPF

### Déterminer la politique SPF

Lister toutes les sources d'envoi légitimes :

```bash
# Serveur mail principal
192.0.2.10 (mail.exemple.fr)

# Services tiers (exemples)
# - Google Workspace: include:_spf.google.com
# - Microsoft 365: include:spf.protection.outlook.com
# - SendGrid: include:sendgrid.net
# - Mailchimp: include:servers.mcsv.net
```

### Créer l'enregistrement SPF

**Configuration simple (serveur uniquement)** :

```dns
exemple.fr. IN TXT "v=spf1 mx ip4:192.0.2.10 -all"
```

**Configuration avec services tiers** :

```dns
exemple.fr. IN TXT "v=spf1 mx ip4:192.0.2.10 include:_spf.google.com -all"
```

**Configuration avec sous-domaines** :

```dns
# Domaine principal
exemple.fr. IN TXT "v=spf1 mx ip4:192.0.2.10 -all"

# Sous-domaine pour newsletter
newsletter.exemple.fr. IN TXT "v=spf1 include:sendgrid.net -all"
```

### Publier l'enregistrement

Utiliser votre interface DNS (Cloudflare, OVH, Gandi, etc.) ou fichier de zone :

```bash
# Exemple avec nsupdate (BIND)
nsupdate -k /etc/bind/keys/Kexemple.fr.key << EOF
server dns.exemple.fr
zone exemple.fr
update add exemple.fr. 3600 IN TXT "v=spf1 mx ip4:192.0.2.10 -all"
send
EOF
```

### Vérifier la publication

```bash
dig TXT exemple.fr +short | grep spf
# Résultat attendu: "v=spf1 mx ip4:192.0.2.10 -all"

# Test avec outil en ligne
curl "https://mxtoolbox.com/api/v1/Lookup/spf?argument=exemple.fr"
```

### Configuration de la vérification SPF (réception)

**Debian/Ubuntu** :

```bash
apt install -y postfix-policyd-spf-python
```

**Rocky/AlmaLinux** :

```bash
dnf install -y pypolicyd-spf
```

**Configuration `/etc/postfix/master.cf`** :

```conf
# Ajouter à la fin du fichier
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/bin/policyd-spf
```

**Configuration `/etc/postfix/main.cf`** :

```conf
# Timeout pour SPF
policyd-spf_time_limit = 3600

# Ajouter à smtpd_recipient_restrictions
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    check_policy_service unix:private/policyd-spf
```

**Configuration `/etc/postfix-policyd-spf-python/policyd-spf.conf`** :

```conf
# Configuration SPF
HELO_reject = False
Mail_From_reject = False

# Logging
defaultSeedOnly = 0

# Header
Header_Type = AR
```

**Redémarrer Postfix** :

```bash
systemctl restart postfix
```

## Configuration DKIM avec OpenDKIM

### Installation

**Debian/Ubuntu** :

```bash
apt install -y opendkim opendkim-tools
```

**Rocky/AlmaLinux** :

```bash
dnf install -y opendkim
```

### Génération des clés

```bash
# Créer la structure de répertoires
mkdir -p /etc/opendkim/keys/${DOMAIN}
cd /etc/opendkim/keys/${DOMAIN}

# Générer la paire de clés
opendkim-genkey -b 2048 -d ${DOMAIN} -s ${SELECTOR}

# Renommer pour clarté
mv ${SELECTOR}.private ${SELECTOR}.private.key
mv ${SELECTOR}.txt ${SELECTOR}.public.txt

# Permissions
chown -R opendkim:opendkim /etc/opendkim
chmod 600 /etc/opendkim/keys/${DOMAIN}/${SELECTOR}.private.key
```

### Publier la clé publique DNS

Afficher la clé publique :

```bash
cat /etc/opendkim/keys/${DOMAIN}/${SELECTOR}.public.txt
```

Résultat (exemple) :

```
default._domainkey IN TXT ( "v=DKIM1; h=sha256; k=rsa; "
	  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArF5Y8qv..."
	  "...suite de la clé..." )  ; ----- DKIM key default for exemple.fr
```

Créer l'enregistrement DNS :

```dns
default._domainkey.exemple.fr. IN TXT (
  "v=DKIM1; h=sha256; k=rsa; "
  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArF5Y8qv...."
)
```

**Important** : Retirer les guillemets et parenthèses inutiles selon votre fournisseur DNS.

Vérification DNS :

```bash
dig TXT default._domainkey.exemple.fr +short
```

### Configuration d'OpenDKIM

**Fichier `/etc/opendkim.conf`** :

```conf
# Logging
LogWhy                  yes
Syslog                  yes
SyslogSuccess           yes

# Mode
Mode                    sv

# Canonicalisation
Canonicalization        relaxed/simple

# Sélecteur
Selector                default

# Socket
Socket                  inet:8891@localhost

# User/Group
UserID                  opendkim:opendkim

# PID
PidFile                 /var/run/opendkim/opendkim.pid

# Paths
KeyTable                /etc/opendkim/KeyTable
SigningTable            /etc/opendkim/SigningTable
ExternalIgnoreList      /etc/opendkim/TrustedHosts
InternalHosts           /etc/opendkim/TrustedHosts

# Signature
SignatureAlgorithm      rsa-sha256
OversignHeaders         From
```

**Fichier `/etc/opendkim/KeyTable`** :

```
default._domainkey.exemple.fr exemple.fr:default:/etc/opendkim/keys/exemple.fr/default.private.key
```

**Fichier `/etc/opendkim/SigningTable`** :

```
*@exemple.fr default._domainkey.exemple.fr
```

**Fichier `/etc/opendkim/TrustedHosts`** :

```
127.0.0.1
::1
localhost
mail.exemple.fr
exemple.fr
*.exemple.fr
192.168.1.0/24
```

### Intégration avec Postfix

**Éditer `/etc/postfix/main.cf`** :

```conf
# Milter configuration
milter_default_action = accept
milter_protocol = 6
smtpd_milters = inet:127.0.0.1:8891
non_smtpd_milters = $smtpd_milters
```

### Démarrage des services

```bash
# Activer et démarrer OpenDKIM
systemctl enable opendkim
systemctl start opendkim

# Vérifier le statut
systemctl status opendkim

# Redémarrer Postfix
systemctl restart postfix

# Vérifier les logs
journalctl -u opendkim -f
```

### Test de signature DKIM

```bash
# Envoyer un email de test
echo "Test DKIM" | mail -s "Test" check-auth@verifier.port25.com

# Vérifier les logs
grep -i dkim /var/log/mail.log
```

## Configuration DMARC avec OpenDMARC

### Installation

**Debian/Ubuntu** :

```bash
apt install -y opendmarc
```

**Rocky/AlmaLinux** :

```bash
dnf install -y opendmarc
```

### Publication de l'enregistrement DMARC

**Phase 1 : Monitoring (4-8 semaines)** :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=none; rua=mailto:dmarc@exemple.fr; ruf=mailto:dmarc-forensic@exemple.fr; fo=1; pct=100"
```

**Vérification** :

```bash
dig TXT _dmarc.exemple.fr +short
```

### Configuration d'OpenDMARC

**Fichier `/etc/opendmarc.conf`** :

```conf
# Authentification
AuthservID mail.exemple.fr

# Socket
Socket inet:8893@localhost

# User
UserID opendmarc:opendmarc

# PID
PidFile /var/run/opendmarc/opendmarc.pid

# Ignore authenticated clients
IgnoreAuthenticatedClients true

# Trust authservice IDs
TrustedAuthservIDs mail.exemple.fr

# Reject failures (démarrer avec false)
RejectFailures false

# Reporting
HistoryFile /var/run/opendmarc/opendmarc.dat

# Logging
Syslog true
SyslogFacility mail
```

### Intégration avec Postfix

**Éditer `/etc/postfix/main.cf`** :

```conf
# Ajouter OpenDMARC à la chaîne de milters
smtpd_milters = inet:127.0.0.1:8891, inet:127.0.0.1:8893
non_smtpd_milters = $smtpd_milters
```

### Configuration de la base de données (optionnel)

Pour la génération de rapports automatiques :

```bash
# Installer MySQL/MariaDB
apt install -y mariadb-server

# Créer la base de données
mysql << EOF
CREATE DATABASE opendmarc;
GRANT ALL PRIVILEGES ON opendmarc.* TO 'opendmarc'@'localhost' IDENTIFIED BY 'mot_de_passe_securise';
FLUSH PRIVILEGES;
EOF

# Importer le schéma
mysql opendmarc < /usr/share/doc/opendmarc/schema.mysql
```

**Mise à jour `/etc/opendmarc.conf`** :

```conf
# Database configuration
StorageBackend mysql
DatabaseServer localhost
DatabasePort 3306
DatabaseUser opendmarc
DatabasePassword mot_de_passe_securise
DatabaseName opendmarc
```

### Script de génération des rapports

Créer `/usr/local/bin/opendmarc-reports.sh` :

```bash
#!/bin/bash

DATE=$(date -d "yesterday" +%Y-%m-%d)
REPORT_EMAIL="dmarc@exemple.fr"

/usr/sbin/opendmarc-reports \
  --dbhost=localhost \
  --dbname=opendmarc \
  --dbuser=opendmarc \
  --dbpass=mot_de_passe_securise \
  --interval=86400 \
  --report-email="${REPORT_EMAIL}" \
  --report-org="exemple.fr"

# Nettoyage des anciennes données (> 30 jours)
mysql -u opendmarc -p'mot_de_passe_securise' opendmarc << EOF
DELETE FROM messages WHERE date < DATE_SUB(NOW(), INTERVAL 30 DAY);
DELETE FROM reporters WHERE date < DATE_SUB(NOW(), INTERVAL 30 DAY);
EOF
```

Rendre exécutable :

```bash
chmod +x /usr/local/bin/opendmarc-reports.sh
```

**Cron quotidien** (`/etc/cron.daily/opendmarc-reports`) :

```bash
#!/bin/bash
/usr/local/bin/opendmarc-reports.sh
```

```bash
chmod +x /etc/cron.daily/opendmarc-reports
```

### Démarrage des services

```bash
# Activer et démarrer OpenDMARC
systemctl enable opendmarc
systemctl start opendmarc

# Vérifier le statut
systemctl status opendmarc

# Redémarrer Postfix
systemctl restart postfix

# Vérifier les logs
journalctl -u opendmarc -f
```

## Configuration ARC avec OpenARC (optionnel)

ARC est recommandé si vous opérez un service de forwarding, liste de diffusion ou passerelle email.

### Installation

**Debian/Ubuntu** :

```bash
apt install -y openarc
```

**Rocky/AlmaLinux** :

```bash
dnf install -y openarc
```

### Génération des clés ARC

```bash
# Réutiliser la même clé que DKIM ou en générer une nouvelle
mkdir -p /etc/openarc/keys
cd /etc/openarc/keys

openssl genrsa -out arc.private 2048
openssl rsa -in arc.private -pubout -out arc.public

# Extraire la clé publique pour DNS
openssl rsa -in arc.private -pubout -outform PEM | grep -v '^-' | tr -d '\n'
```

### Publier la clé ARC DNS

```dns
arc._domainkey.exemple.fr. IN TXT (
  "v=DKIM1; k=rsa; "
  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
)
```

### Configuration OpenARC

**Fichier `/etc/openarc/openarc.conf`** :

```conf
# Mode (s=sign, v=verify, sv=both)
Mode                    sv

# Signing
Domain                  exemple.fr
Selector                arc
KeyFile                 /etc/openarc/keys/arc.private

# Socket
Socket                  inet:8892@localhost

# User
UserID                  openarc:openarc

# PID
PidFile                 /var/run/openarc/openarc.pid

# Canonicalisation
Canonicalization        relaxed/relaxed

# Logging
Syslog                  yes
SyslogSuccess           yes
LogWhy                  yes

# Trusted hosts
InternalHosts           /etc/openarc/internal-hosts
TrustedAuthservIDs      mail.exemple.fr

# Headers to sign
SignHeaders             from,to,subject,date,message-id
```

**Fichier `/etc/openarc/internal-hosts`** :

```
127.0.0.1
::1
localhost
mail.exemple.fr
exemple.fr
192.168.1.0/24
```

### Intégration avec Postfix

**Éditer `/etc/postfix/main.cf`** :

```conf
# Chaîne complète de milters : DKIM → ARC → DMARC
smtpd_milters = inet:127.0.0.1:8891, inet:127.0.0.1:8892, inet:127.0.0.1:8893
non_smtpd_milters = $smtpd_milters
```

### Démarrage

```bash
systemctl enable openarc
systemctl start openarc
systemctl status openarc
systemctl restart postfix
```

## Tests et validation

### Test d'envoi complet

```bash
# Envoyer un email de test
echo "Ceci est un test SPF/DKIM/DMARC" | \
mail -s "Test d'authentification" \
-a "From: test@exemple.fr" \
check-auth@verifier.port25.com
```

Vous recevrez un rapport détaillé par email.

### Vérifier les en-têtes

Envoyer un email à votre propre Gmail et examiner les en-têtes :

```bash
# Dans Gmail: Menu → Afficher l'original
```

Rechercher :

```
SPF: PASS
DKIM: PASS
DMARC: PASS
ARC: PASS (si configuré)
```

### Tests avec swaks

```bash
# Installer swaks
apt install -y swaks  # Debian/Ubuntu
dnf install -y swaks  # Rocky/AlmaLinux

# Test SMTP simple
swaks --to destinataire@exemple.com \
      --from test@exemple.fr \
      --server mail.exemple.fr \
      --auth-user user@exemple.fr \
      --auth-password 'password'
```

### Vérifier les logs

```bash
# Logs généraux
tail -f /var/log/mail.log

# Logs DKIM
journalctl -u opendkim -f

# Logs DMARC
journalctl -u opendmarc -f

# Logs ARC
journalctl -u openarc -f

# Filtrer par email
grep 'test@exemple.fr' /var/log/mail.log
```

### Outils de validation en ligne

```bash
# SPF
curl "https://www.kitterman.com/spf/validate.html?domain=exemple.fr"

# DKIM
# Envoyer email à: autorespond+dkim@dk.elandsys.com

# DMARC
dig TXT _dmarc.exemple.fr +short

# Test complet
# https://www.mail-tester.com/
# https://mxtoolbox.com/SuperTool.aspx
```

## Monitoring et maintenance

### Script de monitoring

Créer `/usr/local/bin/check-mail-auth.sh` :

```bash
#!/bin/bash

DOMAIN="exemple.fr"
LOGFILE="/var/log/mail-auth-check.log"

echo "[$(date)] Début de la vérification" >> "${LOGFILE}"

# Vérifier SPF
echo "=== SPF ===" >> "${LOGFILE}"
dig TXT "${DOMAIN}" +short | grep spf >> "${LOGFILE}"

# Vérifier DKIM
echo "=== DKIM ===" >> "${LOGFILE}"
dig TXT "default._domainkey.${DOMAIN}" +short >> "${LOGFILE}"

# Vérifier DMARC
echo "=== DMARC ===" >> "${LOGFILE}"
dig TXT "_dmarc.${DOMAIN}" +short >> "${LOGFILE}"

# Vérifier services
echo "=== Services ===" >> "${LOGFILE}"
systemctl is-active postfix >> "${LOGFILE}"
systemctl is-active opendkim >> "${LOGFILE}"
systemctl is-active opendmarc >> "${LOGFILE}"

# Statistiques des dernières 24h
echo "=== Statistiques SPF ===" >> "${LOGFILE}"
grep "SPF" /var/log/mail.log | grep -c "pass" >> "${LOGFILE}"
grep "SPF" /var/log/mail.log | grep -c "fail" >> "${LOGFILE}"

echo "[$(date)] Fin de la vérification" >> "${LOGFILE}"
```

```bash
chmod +x /usr/local/bin/check-mail-auth.sh
```

**Cron** (`crontab -e`) :

```cron
# Vérification quotidienne à 6h
0 6 * * * /usr/local/bin/check-mail-auth.sh
```

### Alertes avec fail2ban

```bash
# Installer fail2ban
apt install -y fail2ban

# Créer /etc/fail2ban/filter.d/postfix-dmarc.conf
[Definition]
failregex = ^.*opendmarc.*: DMARC fail.*from=<.*@<HOST>>.*$
ignoreregex =

# Créer /etc/fail2ban/jail.d/postfix-dmarc.local
[postfix-dmarc]
enabled = true
port = smtp
filter = postfix-dmarc
logpath = /var/log/mail.log
maxretry = 10
findtime = 3600
bantime = 86400
```

### Rotation des clés DKIM (tous les 6-12 mois)

```bash
#!/bin/bash
# /usr/local/bin/rotate-dkim.sh

DOMAIN="exemple.fr"
OLD_SELECTOR="default"
NEW_SELECTOR="default$(date +%Y%m)"
KEY_DIR="/etc/opendkim/keys/${DOMAIN}"

# Générer nouvelle clé
cd "${KEY_DIR}"
opendkim-genkey -b 2048 -d "${DOMAIN}" -s "${NEW_SELECTOR}"
chown opendkim:opendkim "${NEW_SELECTOR}.private"
chmod 600 "${NEW_SELECTOR}.private"

echo "Nouvelle clé générée: ${NEW_SELECTOR}"
echo "Publiez cette clé DNS:"
cat "${NEW_SELECTOR}.txt"
echo ""
echo "Après propagation DNS (24-48h), mettez à jour:"
echo "- /etc/opendkim/KeyTable"
echo "- /etc/opendkim/SigningTable"
echo "- Redémarrez OpenDKIM"
echo ""
echo "Après 7 jours, supprimez l'ancienne clé:"
echo "- ${KEY_DIR}/${OLD_SELECTOR}.private"
echo "- Enregistrement DNS: ${OLD_SELECTOR}._domainkey.${DOMAIN}"
```

## Sécurisation avancée

### Firewall

```bash
# UFW (Debian/Ubuntu)
apt install -y ufw

ufw default deny incoming
ufw default allow outgoing
ufw allow ssh
ufw allow 25/tcp   # SMTP
ufw allow 587/tcp  # Submission
ufw allow 465/tcp  # SMTPS
ufw enable

# Firewalld (Rocky/AlmaLinux)
firewall-cmd --permanent --add-service=smtp
firewall-cmd --permanent --add-service=smtp-submission
firewall-cmd --permanent --add-service=smtps
firewall-cmd --reload
```

### Rate limiting

**Éditer `/etc/postfix/main.cf`** :

```conf
# Limiter le taux d'envoi par client
smtpd_client_message_rate_limit = 100
smtpd_client_recipient_rate_limit = 200

# Limiter le nombre de connexions simultanées
smtpd_client_connection_count_limit = 10
smtpd_client_connection_rate_limit = 30

# Délai entre les commandes
smtpd_error_sleep_time = 5s
smtpd_soft_error_limit = 5
smtpd_hard_error_limit = 10
```

### PostScreen (protection anti-spam)

**Éditer `/etc/postfix/master.cf`** :

```conf
# Commenter la ligne SMTP standard
#smtp      inet  n       -       y       -       -       smtpd

# Ajouter PostScreen
smtp      inet  n       -       y       -       1       postscreen
smtpd     pass  -       -       y       -       -       smtpd
dnsblog   unix  -       -       y       -       0       dnsblog
tlsproxy  unix  -       -       y       -       0       tlsproxy
```

**Éditer `/etc/postfix/main.cf`** :

```conf
# PostScreen configuration
postscreen_access_list = permit_mynetworks
postscreen_dnsbl_sites =
    zen.spamhaus.org*2,
    b.barracudacentral.org*2,
    bl.spameatingmonkey.net*1,
    bl.spamcop.net*1
postscreen_dnsbl_threshold = 3
postscreen_dnsbl_action = enforce
postscreen_greet_action = enforce
```

### Certificats SSL/TLS avec Let's Encrypt

```bash
# Installer certbot
apt install -y certbot  # Debian/Ubuntu
dnf install -y certbot  # Rocky/AlmaLinux

# Générer le certificat
certbot certonly --standalone -d mail.exemple.fr

# Lier les certificats dans Postfix
postconf -e "smtpd_tls_cert_file = /etc/letsencrypt/live/mail.exemple.fr/fullchain.pem"
postconf -e "smtpd_tls_key_file = /etc/letsencrypt/live/mail.exemple.fr/privkey.pem"

# Renouvellement automatique
crontab -e
# Ajouter:
0 3 * * * certbot renew --quiet --post-hook "systemctl reload postfix"

# Redémarrer Postfix
systemctl restart postfix
```

## Troubleshooting courant

### Problème : Emails non signés DKIM

**Diagnostic** :

```bash
# Vérifier le statut OpenDKIM
systemctl status opendkim

# Vérifier les logs
journalctl -u opendkim | tail -50

# Tester la connexion socket
telnet localhost 8891

# Vérifier les permissions des clés
ls -la /etc/opendkim/keys/exemple.fr/
```

**Solutions** :

```bash
# Permissions incorrectes
chown -R opendkim:opendkim /etc/opendkim/keys
chmod 600 /etc/opendkim/keys/exemple.fr/*.private.key

# Redémarrer les services
systemctl restart opendkim postfix
```

### Problème : SPF fail malgré configuration correcte

**Diagnostic** :

```bash
# Vérifier l'enregistrement SPF
dig TXT exemple.fr +short

# Tester avec outil externe
curl "https://mxtoolbox.com/SuperTool.aspx?action=spf%3aexemple.fr"

# Vérifier l'IP source dans les logs
grep "SPF fail" /var/log/mail.log | tail -10
```

**Solutions** :

```bash
# Ajouter l'IP manquante dans SPF
# Modifier l'enregistrement DNS pour inclure toutes les IPs d'envoi
```

### Problème : DMARC fail à cause de l'alignement

**Diagnostic** :

```bash
# Examiner les en-têtes d'un email rejeté
grep "DMARC" /var/log/mail.log | grep "fail"

# Analyser les rapports DMARC reçus
```

**Solutions** :

```bash
# Vérifier la cohérence From: et MAIL FROM:
# Assurer que DKIM signe avec d=exemple.fr
# Utiliser adkim=r et aspf=r pour alignement relaxed
```

## Checklist finale

```markdown
### Prérequis

☐ Serveur Linux installé et à jour
☐ Postfix installé et configuré
☐ DNS accessible en écriture
☐ Certificat SSL/TLS configuré

### SPF

☐ Enregistrement SPF publié
☐ Toutes les IPs d'envoi listées
☐ Politique -all configurée
☐ policyd-spf installé et actif
☐ Tests d'envoi réussis

### DKIM

☐ OpenDKIM installé
☐ Clés générées (2048 bits)
☐ Clé publique publiée DNS
☐ KeyTable et SigningTable configurés
☐ Milter intégré à Postfix
☐ Emails signés vérifiés

### DMARC

☐ Enregistrement DMARC publié (p=none initial)
☐ Adresses rua/ruf configurées
☐ OpenDMARC installé et actif
☐ Rapports automatiques configurés
☐ Surveillance des rapports en place

### ARC (optionnel)

☐ OpenARC installé
☐ Clé ARC publiée DNS
☐ Configuration validée
☐ Intégration milter effectuée

### Sécurité

☐ Firewall configuré
☐ TLS/SSL actif
☐ Rate limiting configuré
☐ PostScreen actif
☐ Monitoring en place

### Tests

☐ Test avec check-auth@verifier.port25.com
☐ Test avec mail-tester.com (score > 8/10)
☐ Envoi vers Gmail/Yahoo/Outlook validé
☐ Logs vérifiés sans erreurs

### Documentation

☐ Configuration documentée
☐ Procédures de maintenance rédigées
☐ Contacts et responsabilités définis
☐ Plan de rotation des clés établi
```

## Commandes utiles récapitulatives

```bash
# Vérifier la configuration Postfix
postconf -n

# Tester la configuration Postfix
postfix check

# Recharger Postfix
postfix reload

# Vérifier la queue des emails
mailq
postqueue -p

# Forcer l'envoi de la queue
postqueue -f

# Vérifier les logs en temps réel
tail -f /var/log/mail.log

# Statistiques d'envoi
pflogsumm /var/log/mail.log

# Statut de tous les services
systemctl status postfix opendkim opendmarc openarc

# Tester DNS
dig TXT exemple.fr +short
dig TXT _dmarc.exemple.fr +short
dig TXT default._domainkey.exemple.fr +short

# Envoyer email de test
echo "Test" | mail -s "Sujet" destinataire@example.com
```

## Ressources et références

### Documentation officielle

- **Postfix** : http://www.postfix.org/documentation.html
- **OpenDKIM** : http://www.opendkim.org/
- **OpenDMARC** : http://www.trusteddomain.org/opendmarc/
- **OpenARC** : https://github.com/trusteddomainproject/OpenARC

### Outils de test

- **Port25** : http://www.port25.com/dkim-wizard/
- **Mail Tester** : https://www.mail-tester.com/
- **MXToolbox** : https://mxtoolbox.com/
- **DMARC Analyzer** : https://www.dmarcanalyzer.com/

### RFCs de référence

- **SPF** : RFC 7208
- **DKIM** : RFC 6376
- **DMARC** : RFC 7489
- **ARC** : RFC 8617

Cette configuration complète assure une authentification email robuste et conforme aux standards actuels, protégeant votre domaine contre l'usurpation et améliorant significativement votre délivrabilité.
