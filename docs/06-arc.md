---
title: "ARC - chainage de confiance"
---
# ARC - Authenticated Received Chain

!!! note "deprecated feature"

    À l'heure de cette documentation il y a un draft pour rendre ARC deprecated.

## Introduction à ARC

ARC (Authenticated Received Chain) est un protocole d'authentification email défini dans la RFC 8617 (2019) qui résout les problèmes d'authentification causés par les intermédiaires légitimes qui modifient les emails (forwarding, listes de diffusion, etc.).

### Problématique

Lorsqu'un email traverse des intermédiaires légitimes, les mécanismes d'authentification peuvent être cassés :

```text
Expéditeur original     Intermédiaire          Destinataire final
    (SPF ✓)        →    (forwarding)      →      (SPF ✗)
    (DKIM ✓)            (modifie email)         (DKIM ✗)
                                                 (DMARC ✗)
```

**Exemples de scénarios problématiques** :

1. **Forwarding d'emails** : alice@exemple.fr → bob@entreprise.com → bob@gmail.com
2. **Listes de diffusion** : Ajout de footers, modification du sujet
3. **Passerelles de sécurité** : Scans antivirus, ajout de disclaimers
4. **Services anti-spam** : Réécriture de liens, modifications de contenu

### Objectif d'ARC

ARC permet aux intermédiaires de confiance de :

1. **Préserver les résultats d'authentification** originaux (SPF, DKIM, DMARC)
2. **Créer une chaîne de confiance** entre intermédiaires
3. **Permettre au destinataire final** de valider l'authentification d'origine
4. **Éviter les faux positifs DMARC** causés par des modifications légitimes

## Fonctionnement d'ARC

### Principe de la chaîne

Chaque intermédiaire qui traite l'email ajoute un ensemble d'en-têtes ARC qui :

- Documentent les résultats d'authentification qu'il a observés
- Signent cryptographiquement ces résultats
- Créent un lien avec les en-têtes ARC précédents

```text
Email original (i=1)
     ↓
[Intermédiaire 1 ajoute ARC-*: i=1]
     ↓
[Intermédiaire 2 ajoute ARC-*: i=2]
     ↓
[Intermédiaire 3 ajoute ARC-*: i=3]
     ↓
Destinaire final valide la chaîne complète
```

### Les trois en-têtes ARC

Chaque nœud de la chaîne ajoute **trois en-têtes** identifiés par un index `i` :

#### ARC-Authentication-Results (AAR)

Documente les résultats d'authentification observés par cet intermédiaire.

```text
ARC-Authentication-Results: i=1; mx.exemple.fr;
  spf=pass smtp.mailfrom=alice@exemple.fr;
  dkim=pass (signature was verified) header.d=exemple.fr;
  dmarc=pass header.from=exemple.fr
```

#### ARC-Message-Signature (AMS)

Signature cryptographique du message tel qu'il était reçu par cet intermédiaire.

```text
ARC-Message-Signature: i=1; a=rsa-sha256; c=relaxed/relaxed;
  d=exemple.fr; s=arc-selector; t=1704110400;
  h=from:to:subject:date:message-id;
  bh=base64hash==;
  b=base64signature==
```

#### ARC-Seal (AS)

Scelle la chaîne ARC en signant tous les en-têtes ARC précédents plus les siens.

```text
ARC-Seal: i=1; a=rsa-sha256; cv=none; d=exemple.fr;
  s=arc-selector; t=1704110400;
  b=base64signature==
```

### Instance number (i=)

L'index `i` est séquentiel et commence à 1 :

- Premier intermédiaire : `i=1`
- Deuxième intermédiaire : `i=2`
- Troisième intermédiaire : `i=3`
- etc.

Chaque nouveau saut incrémente `i` de 1.

### Chain Validation (cv=)

Le champ `cv` dans ARC-Seal indique le statut de validation de la chaîne ARC précédente :

| Valeur | Signification                      |
| ------ | ---------------------------------- |
| `none` | Aucune chaîne ARC précédente (i=1) |
| `pass` | Chaîne ARC précédente valide       |
| `fail` | Chaîne ARC précédente invalide     |

**Exemple de chaîne** :

```text
# Premier intermédiaire (i=1)
ARC-Seal: i=1; cv=none; ...

# Deuxième intermédiaire (i=2)
ARC-Seal: i=2; cv=pass; ...  # La chaîne i=1 était valide

# Troisième intermédiaire (i=3)
ARC-Seal: i=3; cv=pass; ...  # La chaîne i=1,2 était valide
```

## Structure des en-têtes ARC

### ARC-Authentication-Results (AAR)

**Format** :

```text
ARC-Authentication-Results: i=N; authserv-id;
  method1=result1 property1.name=value;
  method2=result2 property2.name=value
```

**Exemple complet** :

```text
ARC-Authentication-Results: i=1; mx.entreprise.com;
  arc=none;
  spf=pass smtp.mailfrom=user@exemple.fr smtp.helo=mail.exemple.fr;
  dkim=pass (2048-bit key) header.d=exemple.fr header.i=@exemple.fr
    header.b=ABC123 header.a=rsa-sha256 header.s=default;
  dmarc=pass (policy=reject) header.from=exemple.fr
```

**Champs importants** :

- `i=N` : Numéro d'instance dans la chaîne
- `authserv-id` : Identifiant du serveur d'authentification
- `arc=` : État de la chaîne ARC précédente (none/pass/fail)
- `spf=` : Résultat SPF
- `dkim=` : Résultat DKIM
- `dmarc=` : Résultat DMARC

### ARC-Message-Signature (AMS)

**Format** :

```text
ARC-Message-Signature: i=N; a=algorithm; c=canonicalization;
  d=domain; s=selector; t=timestamp;
  h=header-list;
  bh=body-hash;
  b=signature
```

**Exemple complet** :

```text
ARC-Message-Signature: i=1; a=rsa-sha256; c=relaxed/relaxed;
  d=entreprise.com; s=arc2024; t=1704110400;
  h=from:to:cc:subject:date:message-id:references:in-reply-to:
   mime-version:content-type;
  bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
  b=dBp3y8F5kLvH2TsxJKE9m3WqRnP7CzAqW1vE4NcDxL5Q2M8pB9rT0Zj6Hs3Gu
   7KfY4Vn1Ae8UcL3Dw2Xm9Rt5Pq6J8N
```

**Paramètres** :

- `i=` : Instance number
- `a=` : Algorithme (rsa-sha256, rsa-sha1, ed25519-sha256)
- `c=` : Canonicalisation (relaxed/relaxed, simple/simple, etc.)
- `d=` : Domaine du signataire
- `s=` : Sélecteur de clé
- `t=` : Timestamp Unix
- `h=` : Liste des en-têtes signés
- `bh=` : Hash du corps du message
- `b=` : Signature cryptographique

### ARC-Seal (AS)

**Format** :

```text
ARC-Seal: i=N; a=algorithm; cv=chain-validation;
  d=domain; s=selector; t=timestamp;
  b=signature
```

**Exemple complet** :

```text
ARC-Seal: i=2; a=rsa-sha256; cv=pass; d=relay.com; s=arc-seal;
  t=1704115000;
  b=MqD8x2L7nR9pB5vT4cW1jH6sK3mY0fQ8eG7tN2aC4uP9rV5hZ1oE6iL3wB8kJ
   xN7yF4gT0mS9vR2qD6eC1pL5hK
```

**Paramètres spécifiques** :

- `cv=` : Chain validation (none, pass, fail)
  - `none` : Premier maillon (i=1)
  - `pass` : Chaîne précédente valide
  - `fail` : Chaîne précédente cassée (rare, généralement on n'ajoute pas de maillon)

## Exemple complet d'une chaîne ARC

### Scénario

1. Alice (alice@exemple.fr) envoie un email à bob@entreprise.com
2. Bob a configuré un forwarding vers bob@gmail.com
3. Gmail doit valider l'email malgré le forwarding

### En-têtes après le premier intermédiaire (entreprise.com)

```text
ARC-Authentication-Results: i=1; mx.entreprise.com;
  spf=pass smtp.mailfrom=alice@exemple.fr;
  dkim=pass header.d=exemple.fr header.s=default;
  dmarc=pass header.from=exemple.fr

ARC-Message-Signature: i=1; a=rsa-sha256; c=relaxed/relaxed;
  d=entreprise.com; s=arc; t=1704110400;
  h=from:to:subject:date:message-id:dkim-signature;
  bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
  b=GyHYJJd3xL8mKpQnR7sT9v...

ARC-Seal: i=1; a=rsa-sha256; cv=none; d=entreprise.com;
  s=arc; t=1704110400;
  b=K3mN9pQ2rT5vW8xZ...

Received: from mail.exemple.fr (mail.exemple.fr [192.0.2.1])
  by mx.entreprise.com with ESMTPS id abc123
  for <bob@entreprise.com>; Mon, 1 Jan 2024 12:00:00 +0100
DKIM-Signature: v=1; a=rsa-sha256; d=exemple.fr; s=default;
  h=from:to:subject:date;
  b=OriginalSignature...
From: alice@exemple.fr
To: bob@entreprise.com
Subject: Test ARC
Date: Mon, 1 Jan 2024 11:55:00 +0100

Contenu du message
```

### En-têtes après le deuxième intermédiaire (gmail.com)

```text
ARC-Authentication-Results: i=2; mx.google.com;
  arc=pass (i=1);
  spf=fail (forwarded) smtp.mailfrom=bob@entreprise.com;
  dkim=pass header.d=exemple.fr;
  dmarc=fail (SPF failed, forwarding detected)

ARC-Message-Signature: i=2; a=rsa-sha256; c=relaxed/relaxed;
  d=google.com; s=arc-20240101; t=1704110450;
  h=arc-authentication-results:arc-message-signature:arc-seal:
   from:to:subject:date:message-id;
  bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
  b=TnV8wY2zA5bD9cE3fG...

ARC-Seal: i=2; a=rsa-sha256; cv=pass; d=google.com;
  s=arc-20240101; t=1704110450;
  b=P7qR9sT3uV6wX9yZ...

[En-têtes ARC i=1 de entreprise.com]
[En-têtes originaux]
```

**Analyse par Gmail** :

- SPF échoue (IP de entreprise.com, pas exemple.fr)
- DKIM passe (signature préservée)
- DMARC échouerait normalement
- **MAIS** : ARC montre que l'authentification était valide à l'origine (i=1)
- Gmail peut donc accepter l'email grâce à ARC

## Validation de la chaîne ARC

### Processus de validation

1. **Trouver tous les en-têtes ARC** et les grouper par instance (i=)
2. **Vérifier la séquence** : i doit être continu (1, 2, 3, ..., N)
3. **Pour chaque instance i** (de 1 à N) :
   - Vérifier que les 3 en-têtes sont présents (AAR, AMS, AS)
   - Valider la signature ARC-Message-Signature
   - Valider l'ARC-Seal
4. **Vérifier l'ARC-Seal du dernier maillon** (i=N)
5. **Si tout est valide** : Examiner les résultats d'authentification du premier maillon (i=1)

### Résultats possibles

| Résultat | Signification                                  | Action                           |
| -------- | ---------------------------------------------- | -------------------------------- |
| `pass`   | Chaîne valide, authentification d'origine OK   | Accepter l'email                 |
| `fail`   | Chaîne cassée ou authentification d'origine KO | Appliquer politique locale       |
| `none`   | Pas de chaîne ARC                              | Utiliser SPF/DKIM/DMARC standard |

### Commande de validation

Avec `openarc` (ligne de commande) :

```bash
# Valider un email avec ARC
cat email.eml | openarc -t
```

Avec Python et `dkimpy` :

```python
import dkim
from email import message_from_file

with open('email.eml', 'rb') as f:
    msg = message_from_file(f)

result = dkim.arc_verify(msg.as_bytes())
print(f"ARC validation: {result}")
```

## Configuration DNS pour ARC

ARC utilise le même mécanisme de publication de clés que DKIM.

### Génération de clés

```bash
# Générer une paire de clés RSA 2048 bits
openssl genrsa -out arc-private.key 2048
openssl rsa -in arc-private.key -pubout -out arc-public.key

# Extraire la clé publique en format DNS
openssl rsa -in arc-private.key -pubout -outform PEM | \
  grep -v '^-' | tr -d '\n'
```

### Enregistrement DNS

Format identique à DKIM :

```dns
arc-selector._domainkey.exemple.fr. IN TXT "v=DKIM1; k=rsa; p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA..."
```

**Exemple complet** :

```dns
# Sélecteur pour ARC
arc2024._domainkey.exemple.fr. IN TXT (
  "v=DKIM1; k=rsa; "
  "p=MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEArF5Y8qv..."
)
```

### Vérification DNS

```bash
dig TXT arc2024._domainkey.exemple.fr +short
```

## Implémentation avec OpenARC

### Installation sur Debian/Ubuntu

```bash
apt update
apt install openarc
```

### Installation sur RHEL/Rocky/AlmaLinux

```bash
dnf install epel-release
dnf install openarc
```

### Configuration de base

Fichier `/etc/openarc/openarc.conf` :

```conf
# Mode de fonctionnement
Mode                    sv
# s = sign (signer)
# v = verify (vérifier)
# sv = both

# Signature ARC
Domain                  exemple.fr
Selector                arc2024
KeyFile                 /etc/openarc/keys/arc-private.key

# Socket de communication avec le MTA
Socket                  inet:8893@localhost

# Logging
Syslog                  yes
SyslogSuccess           yes
LogWhy                  yes

# Canonicalisation
Canonicalization        relaxed/relaxed

# En-têtes à signer
SignHeaders             from,to,subject,date,message-id,references,in-reply-to,mime-version

# Tables de confiance
InternalHosts           /etc/openarc/internal-hosts
TrustedAuthservIDs      mx.exemple.fr

# Politique
EnableCopySafe          no
PeerList                /etc/openarc/peer-list
```

### Configuration des hôtes internes

Fichier `/etc/openarc/internal-hosts` :

```text
127.0.0.1
::1
192.168.1.0/24
exemple.fr
*.exemple.fr
```

### Configuration des pairs de confiance

Fichier `/etc/openarc/peer-list` (optionnel) :

```text
gmail.com
outlook.com
yahoo.com
```

### Intégration avec Postfix

Fichier `/etc/postfix/main.cf` :

```conf
# Filtre ARC
smtpd_milters = inet:127.0.0.1:8893
non_smtpd_milters = $smtpd_milters
milter_default_action = accept
milter_protocol = 6
```

**Note** : Si vous utilisez déjà OpenDKIM, vous pouvez chaîner les milters :

```conf
smtpd_milters = inet:127.0.0.1:8891, inet:127.0.0.1:8893
# 8891 = OpenDKIM
# 8893 = OpenARC
```

### Démarrage du service

```bash
# Systemd
systemctl enable openarc
systemctl start openarc
systemctl status openarc

# Vérifier les logs
journalctl -u openarc -f
```

## Cas d'usage et exemples

### Cas 1 : Forwarding simple

**Problème** : SPF casse, DKIM OK

**Sans ARC** :

- SPF: fail (nouvelle IP source)
- DKIM: pass
- DMARC: fail (pas d'alignement SPF)
- Résultat : Email rejeté ou mis en spam

**Avec ARC** :

- Le serveur de forwarding ajoute ARC i=1 avec les résultats d'origine
- Le destinataire final voit que l'email était légitime à l'origine
- Résultat : Email accepté

### Cas 2 : Liste de diffusion

**Problème** : Modification du sujet, ajout de footer → DKIM casse

**Sans ARC** :

- SPF: fail (IP de la liste)
- DKIM: fail (contenu modifié)
- DMARC: fail
- Résultat : Email rejeté

**Avec ARC** :

- La liste de diffusion capture les résultats d'authentification d'origine
- Ajoute un maillon ARC avant de modifier l'email
- Le destinataire peut valider l'origine malgré les modifications
- Résultat : Email accepté

### Cas 3 : Passerelle de sécurité

**Problème** : Scan antivirus, ajout de disclaimer → Signature DKIM cassée

**Solution avec ARC** :

- La passerelle vérifie SPF/DKIM/DMARC
- Ajoute un maillon ARC avec les résultats
- Modifie l'email (scan, disclaimer)
- Signe avec sa propre clé DKIM si configuré
- Le destinataire interne peut valider via ARC

## Limites et considérations

### Adoption limitée

ARC n'est pas encore universellement déployé :

- **Supporté** : Gmail, Yahoo, Outlook.com, Fastmail
- **Support partiel** : Certains fournisseurs d'entreprise
- **Non supporté** : Beaucoup de petits fournisseurs

### Confiance requise

ARC repose sur la confiance envers les intermédiaires :

- L'intermédiaire doit être légitime
- Il doit correctement valider l'authentification avant d'ajouter ARC
- Un intermédiaire malveillant pourrait ajouter des en-têtes ARC frauduleux

### Complexité

ARC ajoute de la complexité :

- Configuration supplémentaire
- Gestion de clés additionnelles
- Débogage plus difficile

### Quand utiliser ARC

ARC est recommandé si vous êtes :

1. **Un fournisseur de service email** (Gmail, Yahoo, etc.)
2. **Une liste de diffusion** (Mailman, Listserv, etc.)
3. **Une passerelle de sécurité** d'entreprise
4. **Un service de forwarding** automatique

ARC n'est généralement **pas nécessaire** pour :

- Serveurs d'envoi simples
- Domaines qui n'envoient qu'à leurs propres utilisateurs
- Environnements sans intermédiaires

## Débogage et validation

### Vérifier les en-têtes ARC

```bash
# Extraire tous les en-têtes ARC
grep -i "^ARC-" email.eml
```

### Valider une chaîne ARC

Avec `dkimpy` (Python) :

```python
#!/usr/bin/env python3
import email
import dkim
import sys

with open(sys.argv[1], 'rb') as f:
    msg = email.message_from_binary_file(f)

# Validation ARC
arc_result = dkim.arc_verify(msg.as_bytes())
print(f"ARC Result: {arc_result}")

# Détails
if arc_result == dkim.ARC_SUCCESS:
    print("✓ Chaîne ARC valide")
elif arc_result == dkim.ARC_FAIL:
    print("✗ Chaîne ARC invalide")
else:
    print("- Pas de chaîne ARC ou incomplète")
```

### Outils en ligne

- **Google Admin Toolbox** : https://toolbox.googleapps.com/apps/messageheader/
- **MXToolbox** : https://mxtoolbox.com/Public/Tools/EmailHeaders.aspx
- **ARC Validator** : Certains services proposent une validation spécifique

### Logs OpenARC

```bash
# Voir les opérations ARC
journalctl -u openarc --since "1 hour ago"

# Chercher les échecs
journalctl -u openarc | grep -i fail

# Mode debug
# Dans /etc/openarc/openarc.conf
Loglevel 7
```

## ARC vs DMARC

### Complémentarité

ARC ne remplace pas DMARC, ils sont complémentaires :

| Aspect        | DMARC                               | ARC                                                       |
| ------------- | ----------------------------------- | --------------------------------------------------------- |
| **Objectif**  | Authentifier l'expéditeur d'origine | Préserver l'authentification à travers les intermédiaires |
| **Portée**    | Expéditeur → Premier destinataire   | Toute la chaîne de transmission                           |
| **Politique** | Définie par l'expéditeur            | Pas de politique, juste documentation                     |
| **Action**    | Rejet/Quarantaine/Aucune            | Information pour décision                                 |

### Interaction

```mermaid
flowchart TD
    A["1. Email envoyé<br/>SPF + DKIM + DMARC valides"]
    --> B["2. Intermédiaire vérifie DMARC"]

    B -->|DMARC Pass| C["3. Intermédiaire ajoute les en-têtes ARC<br/>Capture : DMARC = pass"]

    C --> D["4. Intermédiaire modifie le message<br/>Ex. liste de diffusion, passerelle, transfert"]

    D --> E["La modification invalide la signature DKIM d'origine"]

    E --> F["5. Destinataire final vérifie le message"]

    F --> G["Évaluation DMARC directe"]
    G --> H["SPF/DKIM non alignés ou DKIM invalide"]
    H --> I["DMARC direct : Fail"]

    F --> J["Validation de la chaîne ARC"]
    J --> K["ARC scellé et chaîne valide"]
    K --> L["ARC indique : DMARC = Pass<br/>chez l'intermédiaire"]

    I --> M{"Politique locale du destinataire<br/>fait-elle confiance à ARC ?"}
    L --> M

    M -->|Oui| N["Décision : accepter le message<br/>en s'appuyant sur ARC"]
    M -->|Non| O["Décision : appliquer DMARC direct<br/>reject / quarantine / spam"]
```

## Bonnes pratiques

### Pour les expéditeurs

1. **Configurer SPF, DKIM et DMARC d'abord** : ARC ne sert à rien sans authentification de base
2. **Monitorer les rapports DMARC** : Identifier les cas de forwarding légitimes
3. **Utiliser DKIM avec parcimonie** : Ne pas signer les en-têtes qui pourraient être modifiés

### Pour les intermédiaires

1. **Valider avant d'ajouter ARC** : Ne jamais ajouter ARC sans avoir vérifié l'authentification
2. **Utiliser des clés robustes** : RSA 2048 bits minimum
3. **Rotation des clés** : Changer régulièrement les sélecteurs
4. **Logger les décisions** : Garder une trace des validations ARC

### Pour les destinataires

1. **Configurer la validation ARC** : Utiliser OpenARC ou un service équivalent
2. **Politique de confiance** : Définir quels domaines intermédiaires sont dignes de confiance
3. **Fallback sur DMARC** : Si ARC échoue ou est absent, utiliser DMARC standard
4. **Monitoring** : Surveiller les taux de validation ARC

## Exemple complet d'email avec ARC

```text
Return-Path: <bob@entreprise.com>
Received: from mx.google.com (mx.google.com [203.0.113.50])
  by gmail.com with ESMTPS id xyz789
  for <bob@gmail.com>; Mon, 1 Jan 2024 12:01:00 +0100
ARC-Seal: i=2; a=rsa-sha256; cv=pass; d=google.com; s=arc-20240101;
  t=1704110460; b=MqD8x2L7nR9pB5vT4cW1jH6sK3mY0fQ8eG7tN2aC...
ARC-Message-Signature: i=2; a=rsa-sha256; c=relaxed/relaxed;
  d=google.com; s=arc-20240101; t=1704110460;
  h=arc-authentication-results:arc-message-signature:arc-seal:
   from:to:subject:date:message-id:dkim-signature;
  bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
  b=TnV8wY2zA5bD9cE3fG7hJ9kL1mN4oP6qR8sT0uV2wX...
ARC-Authentication-Results: i=2; mx.google.com;
  arc=pass (i=1 spf=pass dkim=pass dmarc=pass);
  spf=fail (forwarded) smtp.mailfrom=bob@entreprise.com;
  dkim=pass header.d=exemple.fr header.s=default;
  dmarc=fail (SPF failed)
Received: from mx.entreprise.com (mx.entreprise.com [198.51.100.25])
  by mx.google.com with ESMTPS id abc456
  for <bob@gmail.com>; Mon, 1 Jan 2024 12:00:30 +0100
ARC-Seal: i=1; a=rsa-sha256; cv=none; d=entreprise.com;
  s=arc; t=1704110430; b=K3mN9pQ2rT5vW8xZ1yB4cD6eF8gH0iJ2kL4mN6oP...
ARC-Message-Signature: i=1; a=rsa-sha256; c=relaxed/relaxed;
  d=entreprise.com; s=arc; t=1704110430;
  h=from:to:subject:date:message-id:dkim-signature;
  bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
  b=GyHYJJd3xL8mKpQnR7sT9vW1xY3zA5bC7dE9fG1hI...
ARC-Authentication-Results: i=1; mx.entreprise.com;
  spf=pass smtp.mailfrom=alice@exemple.fr smtp.helo=mail.exemple.fr;
  dkim=pass (2048-bit key) header.d=exemple.fr header.s=default;
  dmarc=pass (policy=reject) header.from=exemple.fr
Received: from mail.exemple.fr (mail.exemple.fr [192.0.2.10])
  by mx.entreprise.com with ESMTPS id def789
  for <bob@entreprise.com>; Mon, 1 Jan 2024 12:00:00 +0100
DKIM-Signature: v=1; a=rsa-sha256; c=relaxed/relaxed;
  d=exemple.fr; s=default; t=1704110395;
  h=from:to:subject:date:message-id;
  bh=frcCV1k9oG9oKj3dpUqdJg1PxRT2RSN/XKdLCPjaYaY=;
  b=VwX9yZ1aB3cD5eF7gH9iJ1kL3mN5oP7qR9sT1uV3wX5yZ7a...
From: alice@exemple.fr
To: bob@entreprise.com
Subject: Test avec ARC
Date: Mon, 1 Jan 2024 11:55:00 +0100
Message-ID: <unique-id@exemple.fr>

Contenu du message original.
```

**Analyse** :

1. **i=1** (entreprise.com) : Authentification d'origine OK (SPF, DKIM, DMARC pass)
2. **i=2** (google.com) : ARC validé, DMARC direct échoue mais ARC compense
3. **Résultat** : Gmail accepte l'email grâce à la chaîne ARC valide

## Résumé

**ARC en quelques points** :

1. **Résout les problèmes** de forwarding et de modification légitime
2. **Trois en-têtes par maillon** : AAR, AMS, AS
3. **Chaîne séquentielle** : i=1, i=2, i=3, ...
4. **Complémentaire à DMARC** : Ne le remplace pas
5. **Adoption croissante** : Supporté par les grands fournisseurs
6. **Configuration similaire à DKIM** : Clés DNS, signatures cryptographiques
7. **Utile pour les intermédiaires** : Listes de diffusion, passerelles, services de forwarding

**Quand implémenter ARC** :

- Vous êtes un intermédiaire qui modifie les emails
- Vous opérez une liste de diffusion
- Vous gérez une passerelle de sécurité email
- Vous fournissez un service de forwarding

**Configuration recommandée** :

```bash
# OpenARC avec Postfix
apt install openarc
# Configuration dans /etc/openarc/openarc.conf
# Mode sv (sign + verify)
# Intégration milter avec Postfix
```

ARC est un protocole moderne qui améliore significativement la gestion de l'authentification email dans des scénarios complexes impliquant des intermédiaires légitimes.
