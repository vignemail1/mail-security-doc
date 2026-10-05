---
title: "DNS pour les emails"
---
# Enregistrements DNS pour les Emails

## Rôle du DNS dans l'email

Le DNS (Domain Name System) joue un rôle central dans l'acheminement et l'authentification des emails. Plusieurs types d'enregistrements DNS sont utilisés :

## Types d'enregistrements DNS pertinents

### Enregistrement MX (Mail eXchanger)

**Rôle** : Définit le ou les serveurs responsables de la réception des emails pour un domaine.

**Format** :

```
example.com.  IN  MX  10  mail.example.com.
example.com.  IN  MX  20  mail2.example.com.
```

**Composants** :

- **Priorité** : Plus le chiffre est bas, plus la priorité est élevée (10 avant 20)
- **Nom du serveur** : FQDN du serveur mail (doit pointer vers un enregistrement A ou AAAA)

**Règles importantes** :

- L'enregistrement MX ne doit JAMAIS pointer vers un CNAME
- Il doit pointer vers un nom d'hôte, pas une adresse IP
- Plusieurs MX permettent la redondance

**Exemple complet** :

```
example.com.           IN  MX  10  mail1.example.com.
example.com.           IN  MX  20  mail2.example.com.
mail1.example.com.     IN  A   192.0.2.10
mail2.example.com.     IN  A   192.0.2.20
```

### Enregistrement A et AAAA

**Rôle** : Résolution du nom d'hôte vers une adresse IP.

**Format** :

```
mail.example.com.  IN  A     192.0.2.10
mail.example.com.  IN  AAAA  2001:db8::10
```

- **A** : IPv4
- **AAAA** : IPv6

### Enregistrement PTR (Reverse DNS)

**Rôle** : Résolution inverse IP vers nom d'hôte. Crucial pour la délivrabilité.

**Format** :

```
10.2.0.192.in-addr.arpa.  IN  PTR  mail.example.com.
```

**Importance** :

- Beaucoup de serveurs rejettent les emails sans PTR valide
- Le PTR doit correspondre au HELO/EHLO du serveur
- Configuration généralement effectuée par le fournisseur d'hébergement/FAI

**Vérification** :

```bash
dig -x 192.0.2.10
host 192.0.2.10
```

### Enregistrement TXT

**Rôle** : Stockage de données textuelles arbitraires. Utilisé pour SPF, DKIM, DMARC.

**Format** :

```
example.com.  IN  TXT  "v=spf1 mx ~all"
example.com.  IN  TXT  "v=DMARC1; p=quarantine; rua=mailto:dmarc@example.com"
```

**Contraintes** :

- Longueur maximale par chaîne : 255 caractères
- Longueur totale maximale : 4096 caractères (concaténation de plusieurs chaînes)
- Un domaine peut avoir plusieurs enregistrements TXT différents

**Exemple avec chaînes multiples** :

```
example.com.  IN  TXT  ( "v=DKIM1; k=rsa; "
                         "p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..." )
```

## Enregistrements spécifiques à l'authentification

### SPF (TXT)

Publié à la racine du domaine :

```
example.com.  IN  TXT  "v=spf1 ip4:192.0.2.0/24 include:_spf.google.com ~all"
```

### DKIM (TXT)

Publié sous un sélecteur spécifique :

```
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0G..."
```

**Structure du nom** :

- `selector1` : Nom du sélecteur (choisi par l'administrateur)
- `_domainkey` : Préfixe obligatoire
- `example.com` : Domaine d'envoi

### DMARC (TXT)

Publié sous `_dmarc` :

```
_dmarc.example.com.  IN  TXT  "v=DMARC1; p=reject; rua=mailto:dmarc@example.com"
```

### MTA-STS (TXT et fichier web)

**Rôle** : Force l'utilisation de TLS pour les connexions SMTP entrantes.

**Enregistrement DNS** :

```
_mta-sts.example.com.  IN  TXT  "v=STSv1; id=20260210T120000;"
```

**Fichier de politique** : Hébergé sur `https://mta-sts.example.com/.well-known/mta-sts.txt`

```
version: STSv1
mode: enforce
mx: mail.example.com
max_age: 86400
```

### TLS-RPT (TXT)

**Rôle** : Reporting sur les problèmes TLS.

```
_smtp._tls.example.com.  IN  TXT  "v=TLSRPTv1; rua=mailto:tlsrpt@example.com"
```

## TTL (Time To Live)

Le TTL détermine combien de temps un enregistrement DNS peut être mis en cache.

**Recommandations** :

- **MX** : 3600-86400 secondes (1-24 heures)
- **A/AAAA** : 3600-86400 secondes
- **SPF** : 3600 secondes (pour faciliter les modifications)
- **DKIM** : 3600-86400 secondes
- **DMARC** : 3600 secondes

**Stratégie de modification** :

1. Réduire le TTL à 300 secondes plusieurs heures avant la modification
2. Effectuer la modification
3. Attendre 2x l'ancien TTL
4. Augmenter progressivement le TTL

## Propagation DNS

Après une modification DNS :

- **Propagation théorique** : Durée du TTL
- **Propagation réelle** : Peut prendre 24-48 heures (certains résolveurs ignorent le TTL)
- **Vérification multi-localisations** : Utiliser des outils comme `dnschecker.org`

## Outils de vérification DNS

### En ligne de commande

```bash
# Vérifier les MX
dig example.com MX +short
nslookup -type=MX example.com

# Vérifier les TXT (SPF)
dig example.com TXT +short

# Vérifier DKIM
dig selector1._domainkey.example.com TXT +short

# Vérifier DMARC
dig _dmarc.example.com TXT +short

# Vérifier le PTR
dig -x 192.0.2.10 +short

# Utiliser un serveur DNS spécifique
dig @8.8.8.8 example.com MX
```

### Outils graphiques/web

- **MXToolbox** : https://mxtoolbox.com/
- **DNSChecker** : https://dnschecker.org/
- **Google Admin Toolbox** : https://toolbox.googleapps.com/apps/dig/
- **IntoDNS** : https://intodns.com/

## Erreurs courantes DNS

### MX pointant vers CNAME

**Problème** :

```
example.com.      IN  MX  10  mail.example.com.
mail.example.com. IN  CNAME  hosting.provider.com.
```

**Solution** :

```
example.com.          IN  MX  10  mail.example.com.
mail.example.com.     IN  A   192.0.2.10
```

### Absence de PTR

**Symptôme** : Emails rejetés par Gmail, Outlook, etc.

**Vérification** :

```bash
dig -x $(dig +short mail.example.com)
```

**Solution** : Contacter le fournisseur d'adresse IP pour configurer le PTR.

### Plusieurs enregistrements SPF

**Problème** :

```
example.com.  IN  TXT  "v=spf1 mx ~all"
example.com.  IN  TXT  "v=spf1 include:_spf.google.com ~all"
```

**Résultat** : Échec SPF (seul le premier peut être pris en compte selon le résolveur).

**Solution** : Fusionner en un seul enregistrement :

```
example.com.  IN  TXT  "v=spf1 mx include:_spf.google.com ~all"
```

### TTL trop long

**Problème** : TTL de 86400 ou plus rend les modifications très lentes.

**Solution** : Utiliser 3600 secondes (1 heure) pour les enregistrements d'authentification.

## Sécurité DNS : DNSSEC

**Rôle** : Sécurise les réponses DNS contre la falsification via des signatures cryptographiques.

**Pertinence pour l'email** :

- Empêche l'interception/modification des enregistrements SPF/DKIM/DMARC
- Recommandé mais pas obligatoire
- Certains validateurs DMARC vérifient DNSSEC

**Vérification DNSSEC** :

```bash
dig example.com +dnssec
```

**Activation** : Dépend du registrar et du fournisseur DNS.

## Zone DNS exemple complète

Voici un exemple de zone DNS complète pour un domaine avec tous les enregistrements email :

```zone
$ORIGIN example.com.
$TTL 3600

; Enregistrements de base
@               IN  SOA   ns1.example.com. admin.example.com. (
                          2026021001 ; Serial
                          7200       ; Refresh
                          3600       ; Retry
                          1209600    ; Expire
                          3600 )     ; Minimum TTL

@               IN  NS    ns1.example.com.
@               IN  NS    ns2.example.com.
@               IN  A     192.0.2.5

; Serveurs mail
@               IN  MX    10  mail1.example.com.
@               IN  MX    20  mail2.example.com.
mail1           IN  A     192.0.2.10
mail2           IN  A     192.0.2.20

; SPF
@               IN  TXT   "v=spf1 mx ip4:192.0.2.10 ip4:192.0.2.20 ~all"

; DKIM (plusieurs sélecteurs possibles)
default._domainkey    IN  TXT   "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC..."
202602._domainkey     IN  TXT   "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQD..."

; DMARC
_dmarc          IN  TXT   "v=DMARC1; p=quarantine; sp=quarantine; rua=mailto:dmarc-reports@example.com; ruf=mailto:dmarc-forensic@example.com; fo=1; adkim=r; aspf=r; pct=100; ri=86400"

; MTA-STS
_mta-sts        IN  TXT   "v=STSv1; id=20260210T120000;"
mta-sts         IN  A     192.0.2.30

; TLS-RPT
_smtp._tls      IN  TXT   "v=TLSRPTv1; rua=mailto:tls-reports@example.com"
```

## Prochaines étapes

Maintenant que nous avons compris les fondamentaux DNS, nous pouvons détailler chaque mécanisme d'authentification en commençant par SPF.
