---
title: "Responsabilités de chacun"
---
# Responsabilités des Parties Prenantes

## Introduction

L'authentification email implique plusieurs acteurs qui ont chacun des responsabilités spécifiques. Ce chapitre détaille les rôles et obligations de chaque partie dans l'écosystème de l'authentification email.

## Vue d'ensemble des acteurs

### Schéma des interactions

```mermaid
flowchart TD
    OWNER["Propriétaire du domaine<br/><br/>Définit les politiques :<br/>SPF, DKIM et DMARC"]

    DNS["Enregistrements DNS<br/><br/>• SPF<br/>• Clé publique DKIM<br/>• Politique DMARC"]

    SENDER["Serveur d’envoi<br/>(MTA sortant)"]
    RECEIVER["Serveur récepteur<br/>(MTA entrant)"]

    SEND_ACTIONS["Actions du MTA sortant<br/><br/>• Signe les messages avec DKIM<br/>• Utilise une IP autorisée par SPF<br/>• Transmet l’email"]
    RECEIVE_ACTIONS["Actions du MTA entrant<br/><br/>• Vérifie SPF<br/>• Vérifie DKIM<br/>• Vérifie DMARC<br/>• Applique la politique<br/>• Génère les rapports DMARC"]

    OWNER -->|Publie| DNS
    OWNER -->|Configure| SENDER

    DNS -.->|Enregistrements consultés| RECEIVER

    SENDER -->|Email| RECEIVER
    RECEIVER -->|Rapports DMARC| SENDER

    SENDER --> SEND_ACTIONS
    RECEIVER --> RECEIVE_ACTIONS
```

### Les acteurs principaux

1. **Propriétaire du domaine** : Entité qui possède le nom de domaine
2. **Administrateur DNS** : Gère les enregistrements DNS
3. **Administrateur du serveur d'envoi** : Configure le MTA sortant
4. **Administrateur du serveur récepteur** : Configure le MTA entrant
5. **Fournisseur de services tiers** : SaaS, services d'envoi d'emails
6. **Utilisateurs finaux** : Expéditeurs et destinataires

## Propriétaire du domaine

### Responsabilités

#### Définir la politique d'authentification

**SPF** :

- Identifier toutes les sources d'envoi légitimes
- Définir la politique globale (-all, ~all, ?all)
- Documenter les choix effectués

**DKIM** :

- Décider des sélecteurs à utiliser
- Planifier la rotation des clés
- Définir les en-têtes à signer

**DMARC** :

- Choisir la politique (none, quarantine, reject)
- Définir le rythme de déploiement
- Configurer les adresses de réception des rapports

#### Communiquer avec les parties prenantes

- Informer les administrateurs système des politiques
- Coordonner avec les services tiers (Mailchimp, SendGrid, etc.)
- Documenter les procédures pour les nouveaux services

#### Surveiller et ajuster

- Analyser les rapports DMARC régulièrement
- Identifier les problèmes de délivrabilité
- Ajuster les politiques selon les retours
- Maintenir une documentation à jour

#### Assurer la conformité

- Respecter les exigences des grands fournisseurs (Gmail, Yahoo, etc.)
- Se conformer aux réglementations (RGPD, etc.)
- Protéger la marque contre l'usurpation

### Checklist du propriétaire

```markdown
☐ Inventaire complet des sources d'envoi
☐ Politique SPF définie et documentée
☐ Politique DKIM définie (sélecteurs, rotation)
☐ Politique DMARC progressive planifiée
☐ Adresses email pour rapports DMARC configurées
☐ Processus de revue des rapports établi
☐ Coordination avec les services tiers effectuée
☐ Documentation interne créée
☐ Plan de gestion des incidents préparé
```

## Administrateur DNS

### Responsabilités

#### Publier les enregistrements d'authentification

**SPF** :

```dns
exemple.fr. IN TXT "v=spf1 ip4:192.0.2.10 include:_spf.google.com -all"
```

**DKIM** :

```dns
default._domainkey.exemple.fr. IN TXT "v=DKIM1; k=rsa; p=MIIBIjAN..."
```

**DMARC** :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@exemple.fr"
```

#### Maintenir la qualité des enregistrements

- **Syntaxe correcte** : Vérifier la validité des enregistrements
- **TTL approprié** : 3600s (1h) pour permettre des changements rapides si nécessaire
- **Pas de duplication** : Un seul enregistrement par type et nom
- **Longueur** : Respecter les limites DNS (255 caractères par chaîne, découper si nécessaire)

#### Gérer les rotations de clés

**Processus de rotation DKIM** :

```bash
# Étape 1: Publier la nouvelle clé avec un nouveau sélecteur
default._domainkey.exemple.fr. IN TXT "v=DKIM1; k=rsa; p=OLD_KEY"
default2024._domainkey.exemple.fr. IN TXT "v=DKIM1; k=rsa; p=NEW_KEY"

# Étape 2: Configurer le serveur pour signer avec le nouveau sélecteur
# Attendre propagation DNS (24-48h)

# Étape 3: Retirer l'ancienne clé après une période de grâce (7-14 jours)
```

#### Surveiller la propagation DNS

```bash
# Vérifier la propagation sur plusieurs serveurs DNS
dig @8.8.8.8 TXT _dmarc.exemple.fr +short
dig @1.1.1.1 TXT _dmarc.exemple.fr +short
dig @208.67.222.222 TXT _dmarc.exemple.fr +short
```

#### 5. Assurer la disponibilité

- **Redondance DNS** : Serveurs primaire et secondaire
- **Monitoring** : Alertes en cas d'indisponibilité
- **Backup** : Sauvegardes régulières des zones DNS

### Checklist de l'administrateur DNS

```markdown
☐ Enregistrement SPF publié et validé
☐ Enregistrements DKIM publiés pour tous les sélecteurs
☐ Enregistrement DMARC publié et validé
☐ TTL configurés correctement
☐ Propagation DNS vérifiée
☐ Monitoring DNS en place
☐ Procédure de rotation des clés documentée
☐ Backups des zones DNS effectués
☐ Tests de syntaxe réalisés
```

### Outils recommandés

```bash
# Validation SPF
dig TXT exemple.fr +short | grep spf

# Validation DKIM
dig TXT default._domainkey.exemple.fr +short

# Validation DMARC
dig TXT _dmarc.exemple.fr +short

# Test complet
mxtoolbox.com  # Interface web
```

## Administrateur du serveur d'envoi (MTA sortant)

### Responsabilités

#### Configurer la signature DKIM

**Installation et configuration d'OpenDKIM** :

```bash
# Installation
apt install opendkim opendkim-tools

# Génération des clés
mkdir -p /etc/opendkim/keys/exemple.fr
cd /etc/opendkim/keys/exemple.fr
opendkim-genkey -s default -d exemple.fr
chown -R opendkim:opendkim /etc/opendkim

# Configuration dans /etc/opendkim.conf
Domain exemple.fr
KeyFile /etc/opendkim/keys/exemple.fr/default.private
Selector default
```

#### Respecter les enregistrements SPF

- **Utiliser les IPs autorisées** : S'assurer que le serveur envoie depuis une IP listée dans SPF
- **HELO/EHLO correct** : Utiliser un nom de domaine valide et cohérent
- **MAIL FROM approprié** : Utiliser un domaine qui correspond à la politique SPF

#### Configurer les en-têtes correctement

**En-têtes essentiels** :

```
From: expediteur@exemple.fr
To: destinataire@autre.fr
Subject: Sujet du message
Date: Mon, 1 Jan 2024 12:00:00 +0100
Message-ID: <unique-id@exemple.fr>
```

**Configuration Postfix** :

```conf
# /etc/postfix/main.cf
myhostname = mail.exemple.fr
mydomain = exemple.fr
myorigin = $mydomain
```

#### Gérer la réputation de l'IP

- **Monitoring des listes noires** : Vérifier régulièrement si l'IP est blacklistée
- **Reverse DNS (PTR)** : Configurer correctement
- **Throttling** : Limiter le taux d'envoi pour éviter les blocages
- **Gestion des bounces** : Traiter les retours correctement

```bash
# Vérifier le reverse DNS
dig -x 192.0.2.10 +short
# Devrait retourner: mail.exemple.fr

# Vérifier les blacklists
mxtoolbox.com/blacklists.aspx
```

#### 5. Monitoring et logs

```bash
# Logs Postfix
tail -f /var/log/mail.log | grep -i dkim

# Statistiques d'envoi
pflogsumm /var/log/mail.log

# Vérifier les rejets DMARC
grep -i "dmarc" /var/log/mail.log
```

#### 6. Sécuriser le serveur

- **TLS/SSL** : Activer le chiffrement (opportunistic ou obligatoire)
- **Authentification** : SMTP AUTH pour les clients
- **Rate limiting** : Limiter les abus
- **Firewall** : Restreindre l'accès au port 25

### Checklist de l'administrateur d'envoi

```markdown
☐ OpenDKIM installé et configuré
☐ Clés DKIM générées et sécurisées
☐ Signature DKIM active et testée
☐ En-têtes d'email correctement configurés
☐ HELO/EHLO configuré avec le bon hostname
☐ Reverse DNS (PTR) configuré
☐ TLS/SSL activé
☐ SMTP AUTH configuré
☐ Monitoring des logs en place
☐ Alertes pour rejets DMARC configurées
☐ Vérification des blacklists automatisée
☐ Procédure de gestion des incidents définie
```

### Tests à effectuer

```bash
# Test d'envoi avec vérification DKIM
echo "Test" | mail -s "Test DKIM" -a "From: test@exemple.fr" check-auth@verifier.port25.com

# Vérifier la signature DKIM dans les logs
grep "DKIM-Signature" /var/log/mail.log

# Test SPF
swaks --to test@gmail.com --from test@exemple.fr --server mail.exemple.fr
```

## Administrateur du serveur récepteur (MTA entrant)

### Responsabilités

#### Vérifier SPF

**Configuration dans Postfix** :

```bash
# Installation
apt install postfix-policyd-spf-python

# Configuration dans /etc/postfix/master.cf
policyd-spf  unix  -       n       n       -       0       spawn
    user=policyd-spf argv=/usr/bin/policyd-spf

# Configuration dans /etc/postfix/main.cf
policyd-spf_time_limit = 3600
smtpd_recipient_restrictions =
    permit_mynetworks,
    permit_sasl_authenticated,
    reject_unauth_destination,
    check_policy_service unix:private/policyd-spf
```

#### Vérifier DKIM

**Installation d'OpenDKIM en mode vérification** :

```bash
# Installation
apt install opendkim

# Configuration /etc/opendkim.conf
Mode v
Socket inet:8891@localhost

# Intégration Postfix dans /etc/postfix/main.cf
smtpd_milters = inet:127.0.0.1:8891
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
```

#### Vérifier DMARC

**Installation d'OpenDMARC** :

```bash
# Installation
apt install opendmarc

# Configuration /etc/opendmarc.conf
Socket inet:8893@localhost
IgnoreAuthenticatedClients true
RejectFailures false  # Commencer en mode monitoring

# Intégration Postfix
smtpd_milters = inet:127.0.0.1:8891, inet:127.0.0.1:8893
```

#### Appliquer les politiques DMARC

**Phases de déploiement** :

```conf
# Phase 1: Monitoring uniquement (2-4 semaines)
RejectFailures false
IgnoreAuthenticatedClients true

# Phase 2: Rejet progressif (si politique p=reject)
RejectFailures true
SoftwareHeader true  # Ajoute en-tête sans rejeter

# Phase 3: Application complète
RejectFailures true
SoftwareHeader false
```

#### 5. Générer les rapports DMARC

**Configuration OpenDMARC** :

```conf
# Dans /etc/opendmarc.conf
HistoryFile /var/run/opendmarc/opendmarc.dat

# Script de génération quotidienne
#!/bin/bash
opendmarc-reports -dbhost localhost -dbname opendmarc \
  -dbuser opendmarc -interval 86400
```

**Automatisation avec cron** :

```cron
# /etc/cron.daily/opendmarc-reports
#!/bin/bash
/usr/sbin/opendmarc-reports --dbhost=localhost \
  --dbname=opendmarc --dbuser=opendmarc \
  --interval=86400 --report-email=dmarc-reports@exemple.fr
```

#### 6. Gérer les exceptions

- **Listes blanches** : Pour sources connues et fiables
- **Forwarding légitime** : Utiliser ARC si possible
- **Services internes** : Exceptions pour applications métier

### Checklist de l'administrateur de réception

```markdown
☐ Vérification SPF active
☐ Vérification DKIM active (OpenDKIM)
☐ Vérification DMARC active (OpenDMARC)
☐ Politiques de rejet configurées correctement
☐ Génération de rapports DMARC automatisée
☐ Envoi de rapports DMARC fonctionnel
☐ ARC configuré si nécessaire
☐ Logs de vérification activés
☐ Monitoring des rejets en place
☐ Listes blanches documentées
☐ Procédure d'escalade définie
```

### Monitoring

```bash
# Statistiques SPF
grep "SPF" /var/log/mail.log | grep -c "pass"
grep "SPF" /var/log/mail.log | grep -c "fail"

# Statistiques DKIM
grep "DKIM" /var/log/mail.log | grep -c "pass"
grep "DKIM" /var/log/mail.log | grep -c "fail"

# Statistiques DMARC
grep "DMARC" /var/log/mail.log | grep -c "pass"
grep "DMARC" /var/log/mail.log | grep -c "fail"
```

## Fournisseur de services tiers (SaaS)

### Responsabilités

#### Faciliter la configuration pour les clients

**Documentation à fournir** :

```markdown
# Configuration SPF

Ajouter à votre enregistrement SPF:
include:spf.service-tiers.com

# Configuration DKIM

Ajouter ces enregistrements DNS:
s1.\_domainkey.exemple.fr CNAME s1.domainkey.service-tiers.com
s2.\_domainkey.exemple.fr CNAME s2.domainkey.service-tiers.com

# Configuration DMARC

Recommandations:

- Utiliser un sous-domaine dédié (newsletter.exemple.fr)
- Configurer p=quarantine minimum
```

#### Maintenir une bonne réputation

- **IPs dédiées ou partagées** : Offrir le choix selon les besoins
- **Gestion des plaintes** : Traiter rapidement les abus
- **Compliance** : Respecter les bonnes pratiques anti-spam
- **Monitoring** : Surveiller la réputation des IPs

#### Supporter DKIM

- **Signature automatique** : Tous les emails doivent être signés
- **Gestion des clés** : Rotation régulière et sécurisée
- **Multi-tenant** : Clés séparées par client si possible

#### Fournir des outils de diagnostic

- **Dashboard** : Statistiques de délivrabilité
- **Tests d'envoi** : Outils de vérification SPF/DKIM/DMARC
- **Rapports** : Accès aux données de réputation

### Checklist du fournisseur SaaS

```markdown
☐ Documentation claire pour configuration SPF/DKIM/DMARC
☐ Support de sous-domaines dédiés
☐ Signature DKIM automatique activée
☐ Enregistrements CNAME DKIM disponibles
☐ Monitoring de réputation en place
☐ Dashboard de délivrabilité pour clients
☐ Support technique pour authentification email
☐ Gestion des plaintes automatisée
☐ Conformité aux standards (RFC)
☐ Tests automatisés de configuration
```

### Exemple de configuration client

**Pour Mailchimp** :

```dns
# SPF
newsletter.exemple.fr. IN TXT "v=spf1 include:servers.mcsv.net -all"

# DKIM
k1._domainkey.newsletter.exemple.fr. IN CNAME dkim.mcsv.net.

# DMARC
_dmarc.newsletter.exemple.fr. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@exemple.fr"
```

**Pour SendGrid** :

```dns
# SPF
mail.exemple.fr. IN TXT "v=spf1 include:sendgrid.net -all"

# DKIM
s1._domainkey.mail.exemple.fr. IN CNAME s1.domainkey.u12345.wl.sendgrid.net.
s2._domainkey.mail.exemple.fr. IN CNAME s2.domainkey.u12345.wl.sendgrid.net.

# DMARC
_dmarc.mail.exemple.fr. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@exemple.fr"
```

## Utilisateurs finaux

### Responsabilités des expéditeurs

#### Utiliser les systèmes autorisés

- Envoyer uniquement via les serveurs d'entreprise ou services autorisés
- Ne pas contourner les systèmes de messagerie officiels
- Respecter les politiques d'usage de l'email

#### Signaler les problèmes

- Rapporter les emails légitimes classés en spam
- Signaler les tentatives de phishing
- Alerter sur les problèmes de délivrabilité

### Responsabilités des destinataires

#### Signaler le spam et le phishing

- Utiliser les boutons "Signaler comme spam"
- Ne pas répondre aux emails suspects
- Vérifier l'authenticité des emails sensibles

#### Comprendre les indicateurs d'authentification

**Dans Gmail** :

```
✓ exemple.fr  →  Authentification OK
? via autre.fr  →  Forwarding ou service tiers
⚠ Attention  →  Échec d'authentification
```

## Matrice des responsabilités (RACI)

### Déploiement initial

| Tâche                    | Propriétaire | Admin DNS | Admin Envoi | Admin Réception | SaaS | Utilisateur |
| ------------------------ | ------------ | --------- | ----------- | --------------- | ---- | ----------- |
| Définir politique SPF    | **R**        | C         | C           | I               | C    | -           |
| Publier SPF              | A            | **R**     | I           | I               | -    | -           |
| Configurer DKIM          | A            | C         | **R**       | I               | C    | -           |
| Publier clés DKIM        | A            | **R**     | C           | I               | -    | -           |
| Définir politique DMARC  | **R**        | C         | C           | C               | C    | -           |
| Publier DMARC            | A            | **R**     | I           | I               | -    | -           |
| Configurer vérifications | A            | -         | I           | **R**           | -    | -           |
| Générer rapports         | I            | -         | -           | **R**           | A    | -           |
| Analyser rapports        | **R**        | C         | C           | C               | -    | -           |

**Légende RACI** :

- **R** (Responsible) : Responsable de l'exécution
- **A** (Accountable) : Autorité/Approbateur final
- **C** (Consulted) : Consulté pour avis
- **I** (Informed) : Informé des résultats

### Maintenance continue

| Tâche                   | Propriétaire | Admin DNS | Admin Envoi | Admin Réception | SaaS  |
| ----------------------- | ------------ | --------- | ----------- | --------------- | ----- |
| Analyser rapports DMARC | **R**        | I         | C           | C               | -     |
| Rotation clés DKIM      | A            | **R**     | C           | I               | **R** |
| Ajuster politique SPF   | **R**        | C         | C           | I               | C     |
| Mettre à jour DMARC     | **R**        | C         | I           | I               | -     |
| Monitoring réputation   | C            | -         | **R**       | I               | **R** |
| Traiter incidents       | A            | C         | **R**       | **R**           | C     |
| Documentation           | **R**        | C         | C           | C               | -     |

##  Communication entre parties

Dans la plupart des cas, les parties n'ont pas de relation entre elles sauf dans des cas de troubleshooting dans des cas d'emails filtrer/non délivrés.

### Canaux de communication recommandés

#### Documentation centralisée

Si applicable, par exemple :
- une maquette du stagiaire avec son entreprise, 
- réseau de confiance inter-intreprises,
- délégation compléte d'un sous-domaine à un projet/service spécifique,
- ...

```markdown
# Wiki interne ou confluence

## Configuration actuelle

- Enregistrements SPF/DKIM/DMARC
- Sélecteurs DKIM en usage
- Services tiers autorisés
- IPs d'envoi autorisées

## Procédures

- Ajout d'un nouveau service d'envoi
- Rotation des clés DKIM
- Gestion des incidents
- Escalade

## Contacts

- Propriétaire du domaine: nom@exemple.fr
- Admin DNS: dns-admin@exemple.fr
- Admin messagerie: mail-admin@exemple.fr
```

#### Workflow de changement

```mermaid
1. Demande de changement (ticket)
          ↓
2. Validation propriétaire domaine
          ↓
3. Mise en œuvre (DNS/MTA)
          ↓
4. Tests et validation
          ↓
5. Documentation
          ↓
6. Communication aux parties prenantes
```

#### Réunions régulières

- **Hebdomadaire** : Revue des incidents
- **Mensuelle** : Analyse des rapports DMARC
- **Trimestrielle** : Revue de la politique globale

### Templates de communication

#### Ajout d'un nouveau service tiers

```markdown
Objet: [ACTION REQUISE] Configuration SPF/DKIM pour nouveau service

Bonjour,

Nous allons déployer [Nom du service] pour [usage].

Actions requises:

**Admin DNS:**
☐ Ajouter: include:spf.service.com dans SPF
☐ Ajouter: s1.\_domainkey.exemple.fr CNAME ...

**Admin Envoi:**
☐ Tester l'envoi depuis le service
☐ Vérifier signature DKIM

**Propriétaire:**
☐ Valider dans les rapports DMARC

Date cible: [Date]
Contact: [Nom]
```

##  Gestion des incidents

### Scénarios courants

#### Incident 1: Emails légitimes rejetés

**Responsable principal** : Administrateur d'envoi

**Actions** :

1. Vérifier les logs d'envoi
2. Tester SPF/DKIM/DMARC
3. Analyser les rapports DMARC récents
4. Coordonner avec admin DNS si changement nécessaire
5. Informer propriétaire du domaine

#### Incident 2: Usurpation du domaine détectée

**Responsable principal** : Propriétaire du domaine

**Actions** :

1. Analyser les rapports DMARC forensiques
2. Identifier la source de l'usurpation
3. Renforcer la politique DMARC (p=reject)
4. Communiquer aux utilisateurs
5. Signaler aux autorités si nécessaire

#### Incident 3: Clé DKIM compromise

**Responsable principal** : Administrateur d'envoi

**Actions** :

1. Générer immédiatement nouvelle paire de clés
2. Coordonner avec admin DNS pour publication
3. Basculer sur nouveau sélecteur
4. Révoquer ancienne clé (retirer du DNS)
5. Documenter l'incident
6. Audit de sécurité

##  Conformité et audits

### Checklist d'audit annuel

```markdown
## Configuration DNS

☐ Enregistrement SPF valide et à jour
☐ Toutes les clés DKIM publiées et valides
☐ Enregistrement DMARC avec politique appropriée
☐ Pas d'enregistrements obsolètes
☐ TTL appropriés

## Serveurs d'envoi

☐ Tous les serveurs signent avec DKIM
☐ Toutes les IPs listées dans SPF
☐ Reverse DNS configuré
☐ TLS/SSL actif
☐ Aucune IP blacklistée

## Serveurs de réception

☐ Vérifications SPF/DKIM/DMARC actives
☐ Rapports DMARC générés et envoyés
☐ Politiques appliquées correctement
☐ Logs conservés selon politique de rétention

## Processus

☐ Documentation à jour
☐ Procédures de rotation de clés testées
☐ Procédures d'incident documentées
☐ Formation des équipes effectuée

## Rapports et monitoring

☐ Rapports DMARC analysés régulièrement
☐ Taux de conformité > 95%
☐ Monitoring en place
☐ Alertes configurées
```

### KPIs recommandés

| KPI                            | Cible           | Mesure               |
| ------------------------------ | --------------- | -------------------- |
| Taux de conformité DMARC       | > 95%           | Rapports DMARC       |
| Temps de résolution incident   | < 4h            | Tickets              |
| Rotation clés DKIM             | Tous les 6 mois | Planning             |
| Taux de délivrabilité          | > 98%           | Logs MTA             |
| Emails en spam (faux positifs) | < 0.1%          | Support utilisateurs |

##  Résumé

### Points clés

1. **Propriétaire du domaine** : Définit la stratégie, coordonne, surveille
2. **Admin DNS** : Publie et maintient les enregistrements
3. **Admin envoi** : Configure DKIM, respecte SPF, maintient la réputation
4. **Admin réception** : Vérifie l'authentification, applique politiques, génère rapports
5. **Fournisseur SaaS** : Facilite la configuration, maintient la réputation
6. **Utilisateurs** : Respectent les politiques, signalent les problèmes

### Matrice de décision rapide

| Question                                        | Responsable                     |
| ----------------------------------------------- | ------------------------------- |
| Quelle politique DMARC adopter ?                | Propriétaire du domaine         |
| Comment publier un enregistrement DNS ?         | Administrateur DNS              |
| Comment configurer la signature DKIM ?          | Administrateur d'envoi          |
| Comment traiter les emails qui échouent DMARC ? | Administrateur de réception     |
| Comment configurer un service tiers ?           | Propriétaire + Admin DNS + SaaS |
| Un email légitime est rejeté, que faire ?       | Administrateur d'envoi          |

La réussite du déploiement de l'authentification email repose sur une collaboration étroite et une communication claire entre toutes les parties prenantes.
