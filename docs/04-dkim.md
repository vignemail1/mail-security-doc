---
title: "DKIM - anti-tempering par signature"
---
# DKIM - DomainKeys Identified Mail

## Principe et fonctionnement

DKIM (RFC 6376) utilise la cryptographie asymétrique pour signer numériquement les emails. Cette signature garantit :

1. **Authenticité** : L'email provient bien du domaine revendiqué
2. **Intégrité** : Le contenu n'a pas été modifié en transit

### Flux de signature DKIM

```
┌─────────────────────────────────────────────────────────────┐
│                    Serveur Émetteur                         │
│                                                             │
│  1. Email créé                                              │
│     From: sender@example.com                                │
│     Subject: Test                                           │
│     Body: Hello World                                       │
│                                                             │
│  2. Extraction des en-têtes à signer                        │
│     (From, Subject, Date, ...)                              │
│                                                             │
│  3. Calcul du hash                                          │
│     hash = SHA256(From + Subject + Date + Body)             │
│                                                             │
│  4. Signature avec clé privée                               │
│     signature = RSA_sign(hash, private_key)                 │
│                                                             │
│  5. Ajout de l'en-tête DKIM-Signature                       │
│     DKIM-Signature: v=1; a=rsa-sha256; d=example.com;       │
│                     s=selector1; h=from:subject:date;       │
│                     bh=<body_hash>;                         │
│                     b=<signature>                           │
└─────────────────────────────────────────────────────────────┘
                              │
                              │ Email signé envoyé
                              ▼
┌─────────────────────────────────────────────────────────────┐
│                    Serveur Récepteur                        │
│                                                             │
│  1. Extraction du domaine et sélecteur de DKIM-Signature   │
│     d=example.com, s=selector1                              │
│                                                             │
│  2. Requête DNS                                             │
│     selector1._domainkey.example.com TXT                    │
│                                                             │
│  3. Récupération de la clé publique                         │
│     v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4...         │
│                                                             │
│  4. Vérification de la signature                            │
│     hash_recalculé = SHA256(From + Subject + Date + Body)   │
│     RSA_verify(signature, hash_recalculé, public_key)       │
│                                                             │
│  5. Résultat                                                │
│     Pass / Fail / TempError / PermError                     │
└─────────────────────────────────────────────────────────────┘
```

### Composants DKIM

**Côté émetteur** :

- **Clé privée** : Stockée sur le serveur mail (fichier sécurisé)
- **Logiciel de signature** : OpenDKIM, rspamd, dkimpy-milter

**Côté DNS** :

- **Clé publique** : Publiée dans un enregistrement TXT
- **Format** : `selector._domainkey.domain.com`

**Côté récepteur** :

- **Logiciel de vérification** : Intégré dans la plupart des MTA modernes

## En-tête DKIM-Signature

L'en-tête DKIM-Signature contient tous les paramètres de la signature.

### Exemple complet

```
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
        d=example.com; s=selector1;
        t=1707566400; bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
        h=From:To:Subject:Date:Message-ID;
        b=dzdVyOfAKCdLXdJOc4G2q8D0c8a8fUj/0g2O3ynPy0SV8xN
         kS3Kf3BmO9hD7KlPeP0M2xD8q3K5LpN7O8Q==
```

### Paramètres de DKIM-Signature

| Tag  | Obligatoire | Description                  | Exemple                  |
| ---- | ----------- | ---------------------------- | ------------------------ |
| `v`  | Oui         | Version DKIM                 | `v=1`                    |
| `a`  | Oui         | Algorithme de signature      | `a=rsa-sha256`           |
| `d`  | Oui         | Domaine signataire           | `d=example.com`          |
| `s`  | Oui         | Sélecteur                    | `s=selector1`            |
| `b`  | Oui         | Signature (base64)           | `b=dzdVyOfAK...`         |
| `bh` | Oui         | Hash du corps (base64)       | `bh=frcCV1k9o...`        |
| `h`  | Oui         | En-têtes signés              | `h=from:to:subject:date` |
| `c`  | Non         | Canonicalisation             | `c=relaxed/relaxed`      |
| `t`  | Non         | Timestamp                    | `t=1707566400`           |
| `x`  | Non         | Expiration                   | `x=1707570000`           |
| `l`  | Non         | Longueur du corps signée     | `l=1234`                 |
| `i`  | Non         | Identité de l'agent          | `i=@example.com`         |
| `z`  | Non         | Copie des en-têtes originaux | (rarement utilisé)       |

### Algorithmes de signature

**Supportés** :

- `rsa-sha256` : **Recommandé** (RSA avec SHA-256)
- `rsa-sha1` : Déprécié (moins sécurisé)
- `ed25519-sha256` : Moderne (RFC 8463, support limité)

**Taille de clé RSA** :

- **1024 bits** : Minimum, déprécié
- **2048 bits** : **Recommandé**
- **4096 bits** : Plus sécurisé mais plus lourd (problème avec le DNS)

### Canonicalisation

La canonicalisation définit comment normaliser les en-têtes et le corps avant signature.

**Format** : `c=<header>/<body>`

**Options** :

- `simple` : Aucune modification (strict)
- `relaxed` : Normalisation (espaces, casse, ligne vide)

**Exemples** :

- `c=simple/simple` : Strict pour en-têtes et corps
- `c=relaxed/relaxed` : **Recommandé** (tolère les modifications mineures)
- `c=relaxed/simple` : En-têtes flexibles, corps strict

**Pourquoi relaxed ?**
Les serveurs intermédiaires peuvent modifier légèrement les emails :

- Ajout/suppression d'espaces
- Modification de la casse dans les en-têtes
- Ajout de lignes vides

`relaxed` tolère ces modifications sans invalider la signature.

### En-têtes signés (tag `h`)

La liste des en-têtes inclus dans la signature.

**En-têtes recommandés à signer** :

```
h=from:to:cc:subject:date:message-id:references:in-reply-to
```

**En-têtes à toujours signer** :

- `From` : **Obligatoire** (vérifié par DMARC)
- `Subject`
- `Date`
- `Message-ID`

**En-têtes à ne PAS signer** :

- `Received` : Ajouté par chaque serveur
- `Return-Path` : Modifié lors du transit
- `X-*` : En-têtes personnalisés souvent modifiés

**Attention** : Un en-tête non signé peut être modifié sans invalider DKIM.

## Enregistrement DNS DKIM

### Format de l'enregistrement

**Nom** : `<selector>._domainkey.<domain>`
**Type** : TXT
**Valeur** : `v=DKIM1; k=rsa; p=<public_key>`

**Exemple** :

```
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC5N3lnvvrYgPCRSoqn+awTpkNkRUn7W..."
```

### Tags de l'enregistrement DNS

| Tag | Description                  | Valeur par défaut | Exemple                                  |
| --- | ---------------------------- | ----------------- | ---------------------------------------- |
| `v` | Version                      | DKIM1             | `v=DKIM1`                                |
| `k` | Type de clé                  | rsa               | `k=rsa` ou `k=ed25519`                   |
| `p` | Clé publique (base64)        | Obligatoire       | `p=MIGfMA0GCS...`                        |
| `t` | Flags                        | aucun             | `t=s` (mode strict) ou `t=y` (mode test) |
| `s` | Type de service              | `*` (tous)        | `s=email`                                |
| `h` | Algorithmes de hash acceptés | tous              | `h=sha256`                               |
| `n` | Notes                        | aucune            | `n=Notes for admins`                     |

### Flags importants

**Flag `t=y` (mode test)** :

```
v=DKIM1; k=rsa; t=y; p=MIGfMA0GCS...
```

Signature acceptée même si invalide (pour tests).

**Flag `t=s` (mode strict)** :

```
v=DKIM1; k=rsa; t=s; p=MIGfMA0GCS...
```

L'identité (`i=`) doit correspondre exactement au domaine (`d=`).

### Révocation d'une clé DKIM

Pour révoquer une clé sans supprimer l'enregistrement :

```
selector1._domainkey.example.com.  IN  TXT  "v=DKIM1; p="
```

La clé publique vide (`p=`) invalide toutes les signatures avec ce sélecteur.

## Sélecteurs DKIM

### Qu'est-ce qu'un sélecteur ?

Le sélecteur est un identifiant permettant de publier plusieurs clés DKIM pour un même domaine.

**Avantages** :

- **Rotation des clés** : Publier une nouvelle clé avant de révoquer l'ancienne
- **Séparation des flux** : Une clé par service (marketing, transactionnel, etc.)
- **Multi-serveurs** : Une clé par serveur d'envoi

### Nommage des sélecteurs

**Exemples courants** :

- `default` : Sélecteur par défaut
- `selector1`, `selector2` : Numérotation
- `202602` : Date de rotation (année + mois)
- `marketing`, `transactional` : Par type d'email
- `server1`, `server2` : Par serveur

**Recommandation** : Utiliser la date pour faciliter la rotation.

### Rotation des clés

Processus de rotation tous les 6-12 mois :

**Étape 1** : Créer une nouvelle paire de clés avec un nouveau sélecteur

```bash
# Générer la clé avec sélecteur "202602"
opendkim-genkey -s 202602 -d example.com -b 2048
```

**Étape 2** : Publier la nouvelle clé dans le DNS

```
202602._domainkey.example.com.  IN  TXT  "v=DKIM1; k=rsa; p=MIGfMA0GCS..."
```

**Étape 3** : Attendre la propagation DNS (24-48h)

**Étape 4** : Configurer le serveur pour utiliser le nouveau sélecteur

**Étape 5** : Surveiller pendant 1 semaine

**Étape 6** : Révoquer l'ancienne clé

```
202601._domainkey.example.com.  IN  TXT  "v=DKIM1; p="
```

**Étape 7** : Après 30 jours, supprimer l'ancien enregistrement DNS

## Résultats DKIM

| Résultat      | Description                                       | Action                    |
| ------------- | ------------------------------------------------- | ------------------------- |
| **pass**      | Signature valide                                  | Accepter                  |
| **fail**      | Signature invalide                                | Rejeter ou marquer        |
| **policy**    | Signature valide mais non conforme à la politique | Vérifier alignement DMARC |
| **neutral**   | Signature ne peut pas être évaluée                | Accepter                  |
| **temperror** | Erreur temporaire DNS                             | Réessayer                 |
| **permerror** | Erreur permanente (clé invalide, DNS)             | Rejeter ou marquer        |
| **none**      | Pas de signature DKIM                             | Vérifier SPF/DMARC        |

## Génération des clés DKIM

### Avec OpenDKIM

```bash
# Installation
sudo apt install opendkim-tools   # Debian/Ubuntu
sudo dnf install opendkim-tools   # RHEL/Rocky

# Génération de la paire de clés
opendkim-genkey -b 2048 -d example.com -s selector1

# Fichiers créés :
# - selector1.private : clé privée (à protéger)
# - selector1.txt     : clé publique (pour le DNS)
```

**Contenu de `selector1.txt`** :

```
selector1._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC5N3lnvvrYgPCRSoqn+awTpkNkRUn7W..."
```

### Avec OpenSSL

```bash
# Génération de la clé privée
openssl genrsa -out private.key 2048

# Extraction de la clé publique
openssl rsa -in private.key -pubout -out public.key

# Conversion en format DKIM (retirer les lignes BEGIN/END et les retours à la ligne)
grep -v "BEGIN\|END" public.key | tr -d '\n'
```

### Sécurisation de la clé privée

```bash
# Déplacer la clé dans un répertoire sécurisé
sudo mkdir -p /etc/opendkim/keys/example.com
sudo mv selector1.private /etc/opendkim/keys/example.com/

# Permissions restrictives
sudo chown opendkim:opendkim /etc/opendkim/keys/example.com/selector1.private
sudo chmod 600 /etc/opendkim/keys/example.com/selector1.private

# Vérification
ls -l /etc/opendkim/keys/example.com/selector1.private
# Résultat attendu : -rw------- 1 opendkim opendkim
```

## Problèmes courants DKIM

### Clé publique trop longue

**Problème** : Les clés RSA 4096 bits dépassent la limite DNS (255 caractères par chaîne).

**Solution** : Découper en plusieurs chaînes :

```
selector1._domainkey.example.com. IN TXT (
  "v=DKIM1; k=rsa; "
  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
  "  ...suite de la clé..."
)
```

Ou utiliser RSA 2048 bits (recommandé).

### Signature invalide après modification du message

**Causes** :

- Serveur intermédiaire modifie le corps (ajout de disclaimer, antivirus)
- Liste de diffusion ajoute un footer
- Changement d'encoding (quoted-printable, base64)

**Solution** : Utiliser `c=relaxed/relaxed`.

### En-tête From non signé

**Problème** : DMARC échoue si l'en-tête `From` n'est pas dans le tag `h`.

**Vérification** :

```
DKIM-Signature: ... h=to:subject:date; ...
```

**Solution** : S'assurer que `from` est toujours dans la liste `h`.

### Erreur de requête DNS

**Symptôme** : `temperror` ou `permerror`

**Causes** :

- Enregistrement DNS absent ou mal formé
- DNSSEC invalide
- Serveur DNS injoignable

**Vérification** :

```bash
dig selector1._domainkey.example.com TXT
```

### Multiple signatures DKIM

Un email peut avoir plusieurs signatures DKIM (multi-domaines, services tiers).

**Exemple** :

```
DKIM-Signature: v=1; d=example.com; s=selector1; ...
DKIM-Signature: v=1; d=mailchimp.com; s=k1; ...
```

**DMARC** : Seule la signature alignée avec le domaine `From` compte.

## DKIM et listes de diffusion

Les listes de diffusion posent problème pour DKIM :

1. **Modification du sujet** : `[Liste] Sujet original`
2. **Ajout de footer** : "Unsubscribe: http://..."
3. **Changement du From** : `sender@example.com` → `sender@example.com via list@mailing.org`

**Conséquence** : Signature DKIM invalide.

**Solutions** :

- **ARC (Authenticated Received Chain)** : Préserve les résultats d'authentification
- **DKIM replay** : La liste re-signe l'email avec son propre domaine
- `c=relaxed/relaxed` : Tolère certaines modifications

## DKIM pour sous-domaines

Les sous-domaines doivent avoir leurs propres clés DKIM.

**Exemple** :

```
# Domaine principal
selector1._domainkey.example.com. IN TXT "v=DKIM1; k=rsa; p=MIGfMA0GCS..."

# Sous-domaine
selector1._domainkey.mail.example.com. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgk..."
```

**Politique** : Il n'y a pas d'héritage des clés DKIM.

## Avantages et limites de DKIM

### Avantages

- Garantit l'intégrité du contenu
- Résiste au transfert d'emails (contrairement à SPF)
- Permet la signature par des services tiers
- Clés révocables et rotables

### Limites

- Complexité de mise en place (gestion des clés)
- Vulnérable aux modifications du corps (si `c=simple`)
- Ne garantit pas que le domaine `d=` correspond au `From` visible (rôle de DMARC)
- Taille des clés peut poser problème avec DNS

## Vérification DKIM

### En ligne

- **DKIMValidator** : https://dkimvalidator.com/
- **Mail-Tester** : https://www.mail-tester.com/
- **MXToolbox DKIM** : https://mxtoolbox.com/dkim.aspx

### En ligne de commande

```bash
# Vérifier l'enregistrement DNS
dig selector1._domainkey.example.com TXT +short

# Tester la signature d'un email
python3 -c "import dkim; print(dkim.verify(open('email.eml', 'rb').read()))"
```

### Analyser les en-têtes reçus

```
Authentication-Results: mx.google.com;
       dkim=pass header.i=@example.com header.s=selector1 header.b=dzdVyOfA
```

## DKIM et services tiers

### Mailchimp

Mailchimp signe automatiquement les emails avec sa propre clé :

```
DKIM-Signature: v=1; d=mailchimp.com; s=k1; ...
```

Pour utiliser votre domaine :

1. Générer une clé DKIM dans Mailchimp
2. Publier la clé dans votre DNS
3. Vérifier dans Mailchimp

### SendGrid

Démarche similaire avec authentification du domaine.

### Office 365

DKIM activable dans le portail admin avec génération automatique des clés.

## Récapitulatif DKIM

**Forces** :

- Signature cryptographique robuste
- Garantit l'intégrité
- Fonctionne avec le transfert d'emails
- Permet la rotation des clés

**Faiblesses** :

- Complexité technique
- Gestion des clés privées
- Vulnérable aux modifications du corps (selon canonicalisation)
- Ne lie pas le domaine signataire au `From` visible

**Bonnes pratiques** :

1. Utiliser RSA 2048 bits
2. Algorithme `rsa-sha256`
3. Canonicalisation `relaxed/relaxed`
4. Toujours signer l'en-tête `From`
5. Rotation des clés tous les 6-12 mois
6. Protéger les clés privées (chmod 600)
7. Utiliser un sélecteur basé sur la date
8. Publier la nouvelle clé avant de révoquer l'ancienne
