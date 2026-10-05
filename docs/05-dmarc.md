---
title: "DMARC - reporting et politique"
---
# DMARC - Domain-based Message Authentication, Reporting and Conformance

## Introduction à DMARC

DMARC (Domain-based Message Authentication, Reporting and Conformance) est un protocole d'authentification email publié en 2015 (RFC 7489) qui s'appuie sur SPF et DKIM pour fournir une couche supplémentaire de protection contre l'usurpation d'identité (spoofing) et le phishing.

### Objectifs de DMARC

- **Alignement** : Vérifier que le domaine visible par l'utilisateur (From:) correspond au domaine authentifié par SPF ou DKIM
- **Politique** : Permettre au propriétaire du domaine de spécifier comment traiter les emails qui échouent aux vérifications
- **Rapports** : Recevoir des rapports détaillés sur l'utilisation de votre domaine dans les emails
- **Visibilité** : Obtenir une vue d'ensemble de tous les emails envoyés au nom de votre domaine

### Pourquoi DMARC est nécessaire

SPF et DKIM seuls présentent des limitations :

- **SPF** : Protège l'enveloppe (MAIL FROM) mais pas l'en-tête From: visible par l'utilisateur
- **DKIM** : Signe le message mais ne garantit pas que le domaine signataire correspond au domaine de l'expéditeur visible

DMARC comble ces lacunes en introduisant le concept d'**alignement de l'identifiant**.

## Fonctionnement de DMARC

### Processus de vérification DMARC

```mermaid
flowchart TD
    A["1. Email reçu<br/>From: user@exemple.fr"]
    --> B["2. Vérification SPF"]

    B --> C["3. Vérification DKIM"]

    C --> D["4. Récupération de la politique DMARC<br/>DNS TXT : _dmarc.exemple.fr"]

    D --> E["5. Vérification de l’alignement"]

    E --> SPF["Alignement SPF<br/>Domaine MAIL FROM ↔ domaine From:"]
    E --> DKIM["Alignement DKIM<br/>Domaine DKIM d= ↔ domaine From:"]

    SPF --> F["6. Évaluation DMARC<br/>SPF aligné OU DKIM aligné ?"]
    DKIM --> F

    F -->|Oui| PASS["DMARC Pass<br/>Message accepté"]
    F -->|Non| POLICY["Application de la politique DMARC"]

    POLICY --> P_NONE["p=none<br/>Surveillance / livraison normale"]
    POLICY --> P_QUARANTINE["p=quarantine<br/>Mise en quarantaine / spam"]
    POLICY --> P_REJECT["p=reject<br/>Rejet du message"]

    PASS --> REPORT["7. Génération de rapports<br/>(si rua / ruf configurés)"]
    P_NONE --> REPORT
    P_QUARANTINE --> REPORT
    P_REJECT --> REPORT
```

### Concept d'alignement

L'alignement est le cœur de DMARC. Il existe deux types :

#### Alignement SPF

Le domaine utilisé dans l'enveloppe MAIL FROM doit correspondre au domaine dans l'en-tête From:.

**Alignement strict** :

```
MAIL FROM: <sender@exemple.fr>
From: user@exemple.fr
✓ Aligné (domaines identiques)

MAIL FROM: <sender@mail.exemple.fr>
From: user@exemple.fr
✗ Non aligné (sous-domaines différents)
```

**Alignement relaxed** :

```
MAIL FROM: <sender@mail.exemple.fr>
From: user@exemple.fr
✓ Aligné (même domaine organisationnel)
```

#### Alignement DKIM

Le domaine dans la signature DKIM (d=) doit correspondre au domaine dans l'en-tête From:.

**Alignement strict** :

```
DKIM-Signature: d=exemple.fr
From: user@exemple.fr
✓ Aligné

DKIM-Signature: d=mail.exemple.fr
From: user@exemple.fr
✗ Non aligné
```

**Alignement relaxed** :

```
DKIM-Signature: d=mail.exemple.fr
From: user@exemple.fr
✓ Aligné (même domaine organisationnel)
```

### Condition de validation DMARC

Pour qu'un email passe la vérification DMARC :

- **Au moins un** des deux mécanismes (SPF ou DKIM) doit être aligné ET valide
- Si SPF passe et est aligné → DMARC passe
- Si DKIM passe et est aligné → DMARC passe
- Si les deux échouent ou ne sont pas alignés → DMARC échoue

## Structure d'un enregistrement DMARC

### Emplacement DNS

L'enregistrement DMARC est publié dans un enregistrement TXT DNS au nom :

```
_dmarc.exemple.fr
```

### Syntaxe de base

```
v=DMARC1; p=none; rua=mailto:dmarc@exemple.fr
```

### Balises DMARC

#### Balises obligatoires

| Balise | Description               | Valeurs                        | Exemple        |
| ------ | ------------------------- | ------------------------------ | -------------- |
| `v`    | Version du protocole      | `DMARC1`                       | `v=DMARC1`     |
| `p`    | Politique pour le domaine | `none`, `quarantine`, `reject` | `p=quarantine` |

#### Balises recommandées

| Balise | Description                       | Valeurs               | Exemple                                |
| ------ | --------------------------------- | --------------------- | -------------------------------------- |
| `rua`  | Adresse pour rapports agrégés     | URI mailto: ou https: | `rua=mailto:dmarc-agg@exemple.fr`      |
| `ruf`  | Adresse pour rapports forensiques | URI mailto: ou https: | `ruf=mailto:dmarc-forensic@exemple.fr` |

#### Balises optionnelles

| Balise  | Description                                | Valeurs                        | Défaut        | Exemple     |
| ------- | ------------------------------------------ | ------------------------------ | ------------- | ----------- |
| `sp`    | Politique pour les sous-domaines           | `none`, `quarantine`, `reject` | Valeur de `p` | `sp=reject` |
| `adkim` | Mode d'alignement DKIM                     | `r` (relaxed), `s` (strict)    | `r`           | `adkim=s`   |
| `aspf`  | Mode d'alignement SPF                      | `r` (relaxed), `s` (strict)    | `r`           | `aspf=r`    |
| `pct`   | Pourcentage d'emails soumis à la politique | 0-100                          | 100           | `pct=50`    |
| `fo`    | Options de rapport forensique              | `0`, `1`, `d`, `s`             | `0`           | `fo=1`      |
| `rf`    | Format des rapports forensiques            | `afrf`, `iodef`                | `afrf`        | `rf=afrf`   |
| `ri`    | Intervalle de rapport (secondes)           | Entier positif                 | 86400         | `ri=3600`   |

### Détails des politiques

#### p=none (Surveillance)

```
v=DMARC1; p=none; rua=mailto:dmarc@exemple.fr
```

- **Action** : Aucune action, les emails sont délivrés normalement
- **Usage** : Phase d'apprentissage et de surveillance
- **Durée recommandée** : 2-4 semaines minimum
- **Avantage** : Permet de collecter des données sans risque

#### p=quarantine (Mise en quarantaine)

```
v=DMARC1; p=quarantine; pct=10; rua=mailto:dmarc@exemple.fr
```

- **Action** : Les emails qui échouent sont marqués comme spam ou mis en quarantaine
- **Usage** : Phase de transition
- **Conseil** : Commencer avec `pct=10` puis augmenter progressivement
- **Avantage** : Les emails légitimes peuvent souvent être récupérés

#### p=reject (Rejet)

```
v=DMARC1; p=reject; rua=mailto:dmarc@exemple.fr
```

- **Action** : Les emails qui échouent sont rejetés (non délivrés)
- **Usage** : Protection maximale
- **Prérequis** : Surveillance prolongée en mode `none` ou `quarantine`
- **Avantage** : Protection optimale contre l'usurpation d'identité

## Exemples d'enregistrements DMARC

### Configuration minimale (monitoring)

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=none; rua=mailto:dmarc-reports@exemple.fr"
```

### Configuration intermédiaire

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc-agg@exemple.fr; ruf=mailto:dmarc-forensic@exemple.fr; fo=1"
```

### Configuration stricte

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s; rua=mailto:dmarc@exemple.fr; ruf=mailto:dmarc-forensic@exemple.fr"
```

### Configuration avec alignement relaxed

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; adkim=r; aspf=r; rua=mailto:dmarc@exemple.fr; pct=100; ri=86400"
```

### Configuration pour sous-domaines

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; sp=reject; rua=mailto:dmarc@exemple.fr"
```

Ici :

- Domaine principal (`exemple.fr`) : quarantine
- Sous-domaines (`*.exemple.fr`) : reject

### Configuration pour domaines non-email

Pour les domaines qui n'envoient JAMAIS d'emails :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject; sp=reject; adkim=s; aspf=s"
```

## Rapports DMARC

### Types de rapports

#### Rapports agrégés (RUA)

**Fréquence** : Quotidienne (par défaut)

**Format** : XML compressé (gzip)

**Contenu** :

- Volume d'emails reçus
- Résultats SPF et DKIM
- Alignement
- Adresses IP sources
- Disposition appliquée

**Exemple de structure XML** :

```xml
<?xml version="1.0"?>
<feedback>
  <report_metadata>
    <org_name>gmail.com</org_name>
    <email>noreply-dmarc-support@google.com</email>
    <report_id>12345678901234567890</report_id>
    <date_range>
      <begin>1704067200</begin>
      <end>1704153599</end>
    </date_range>
  </report_metadata>
  <policy_published>
    <domain>exemple.fr</domain>
    <adkim>r</adkim>
    <aspf>r</aspf>
    <p>none</p>
    <sp>none</sp>
    <pct>100</pct>
  </policy_published>
  <record>
    <row>
      <source_ip>192.0.2.1</source_ip>
      <count>142</count>
      <policy_evaluated>
        <disposition>none</disposition>
        <dkim>pass</dkim>
        <spf>pass</spf>
      </policy_evaluated>
    </row>
    <identifiers>
      <header_from>exemple.fr</header_from>
    </identifiers>
    <auth_results>
      <dkim>
        <domain>exemple.fr</domain>
        <result>pass</result>
      </dkim>
      <spf>
        <domain>exemple.fr</domain>
        <result>pass</result>
      </spf>
    </auth_results>
  </record>
</feedback>
```

#### Rapports forensiques (RUF)

**Fréquence** : En temps réel (à chaque échec)

**Format** : Email avec message complet ou extraits

**Contenu** :

- Copie du message ou en-têtes
- Détails de l'échec
- Informations de débogage

**Attention** : Les rapports forensiques peuvent contenir des données sensibles et sont rarement envoyés par les grands fournisseurs de messagerie pour des raisons de confidentialité.

### Options de rapport forensique (fo)

| Valeur | Description                                        |
| ------ | -------------------------------------------------- |
| `fo=0` | Génère un rapport si SPF ET DKIM échouent (défaut) |
| `fo=1` | Génère un rapport si SPF OU DKIM échoue            |
| `fo=d` | Génère un rapport si DKIM échoue                   |
| `fo=s` | Génère un rapport si SPF échoue                    |

Exemple :

```
v=DMARC1; p=quarantine; ruf=mailto:forensic@exemple.fr; fo=1
```

### Analyse des rapports

#### Outils d'analyse

- **parsedmarc** : Outil Python pour parser et analyser les rapports
- **DMARC Analyzer** : Solutions SaaS (Dmarcian, Valimail, etc.)
- **ELK Stack** : Pour centraliser et visualiser les rapports
- **Postmark** : Service gratuit d'analyse DMARC

#### Métriques importantes

1. **Taux de conformité DMARC** : % d'emails qui passent DMARC
2. **Sources d'envoi** : Identifier toutes les IP envoyant pour votre domaine
3. **Échecs d'alignement** : Identifier les problèmes de configuration
4. **Tentatives d'usurpation** : Détecter les utilisations frauduleuses du domaine

## Déploiement progressif de DMARC

### Phase 1 : Monitoring (4-8 semaines)

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=none; rua=mailto:dmarc@exemple.fr"
```

**Objectifs** :

- Identifier toutes les sources d'envoi légitimes
- Détecter les problèmes de configuration SPF/DKIM
- Comprendre le volume et les patterns d'envoi

**Actions** :

- Analyser les rapports quotidiennement
- Corriger les problèmes d'alignement
- Documenter toutes les sources légitimes

### Phase 2 : Quarantaine progressive (4-8 semaines)

```dns
# Semaine 1-2
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; pct=10; rua=mailto:dmarc@exemple.fr"

# Semaine 3-4
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; pct=25; rua=mailto:dmarc@exemple.fr"

# Semaine 5-6
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; pct=50; rua=mailto:dmarc@exemple.fr"

# Semaine 7-8
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; pct=100; rua=mailto:dmarc@exemple.fr"
```

**Objectifs** :

- Tester l'impact de la quarantaine progressivement
- Minimiser les faux positifs
- Affiner les configurations SPF/DKIM

**Actions** :

- Surveiller les plaintes des utilisateurs
- Vérifier que les emails légitimes ne sont pas bloqués
- Augmenter `pct` graduellement

### Phase 3 : Rejet (après validation complète)

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc@exemple.fr"
```

**Objectifs** :

- Protection maximale
- Blocage des tentatives d'usurpation

**Prérequis** :

- Taux de conformité > 95%
- Aucun problème détecté en quarantine pendant 4 semaines
- Toutes les sources légitimes identifiées et configurées

## Gestion des sous-domaines

### Politique pour sous-domaines (sp)

Par défaut, les sous-domaines héritent de la politique du domaine parent. La balise `sp` permet de définir une politique spécifique.

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=quarantine; sp=reject; rua=mailto:dmarc@exemple.fr"
```

Ici :

- `exemple.fr` : quarantine
- `mail.exemple.fr`, `app.exemple.fr`, etc. : reject

### Enregistrement DMARC par sous-domaine

Il est possible de définir une politique spécifique pour un sous-domaine :

```dns
_dmarc.mail.exemple.fr. IN TXT "v=DMARC1; p=none; rua=mailto:dmarc-mail@exemple.fr"
_dmarc.app.exemple.fr. IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc-app@exemple.fr"
```

### Stratégie recommandée

1. **Domaine principal** : Politique progressive (none → quarantine → reject)
2. **Sous-domaines d'envoi actifs** : Politique spécifique ajustée
3. **Sous-domaines inactifs** : `p=reject` immédiat
4. **Sous-domaines génériques** : Utiliser `sp=reject` sur le domaine parent

## Cas d'usage complexes

### Forwarding d'emails

Le forwarding pose problème pour SPF (l'IP change) mais pas pour DKIM (signature préservée).

**Solution** :

- S'assurer que DKIM est configuré et aligné
- Utiliser `adkim=r` pour alignement relaxed
- Considérer SRS (Sender Rewriting Scheme) pour SPF

### Listes de diffusion

Les listes de diffusion modifient souvent les emails, cassant DKIM.

**Solutions** :

- Configurer la liste pour préserver les signatures DKIM
- Utiliser ARC (Authenticated Received Chain)
- Mode `p=quarantine` plutôt que `p=reject`

### Services tiers (SaaS)

Services d'envoi comme Mailchimp, SendGrid, etc.

**Recommandations** :

1. Utiliser un sous-domaine dédié : `newsletter.exemple.fr`
2. Configurer DKIM via le service tiers
3. Ajouter les IPs du service dans SPF
4. Politique DMARC spécifique pour le sous-domaine

**Exemple** :

```dns
# SPF pour le sous-domaine
newsletter.exemple.fr. IN TXT "v=spf1 include:sendgrid.net -all"

# DKIM configuré via SendGrid
s1._domainkey.newsletter.exemple.fr. IN CNAME s1.domainkey.u12345.wl.sendgrid.net.

# DMARC pour le sous-domaine
_dmarc.newsletter.exemple.fr. IN TXT "v=DMARC1; p=quarantine; rua=mailto:dmarc@exemple.fr"
```

## Erreurs courantes et solutions

### Erreur 1 : Pas d'alignement

**Problème** :

```
From: user@exemple.fr
MAIL FROM: <notification@alerts.exemple.fr>
```

SPF passe pour `alerts.exemple.fr` mais n'est pas aligné avec `exemple.fr`.

**Solution** :

- Configurer DKIM avec `d=exemple.fr`
- OU utiliser `From: user@alerts.exemple.fr`
- OU configurer SPF pour inclure les IP et utiliser `adkim=r`

### Erreur 2 : Multiple enregistrements DMARC

**Problème** :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=none"
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject"
```

**Solution** :
Un seul enregistrement DMARC par domaine. Fusionner en un seul :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject; rua=mailto:dmarc@exemple.fr"
```

### Erreur 3 : Politique trop stricte trop tôt

**Problème** :
Passer directement à `p=reject` sans monitoring.

**Solution** :
Suivre le déploiement progressif (none → quarantine → reject) avec analyse des rapports à chaque étape.

### Erreur 4 : Oublier les sous-domaines

**Problème** :
Domaine principal protégé mais sous-domaines vulnérables.

**Solution** :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject; sp=reject; rua=mailto:dmarc@exemple.fr"
```

## DMARC et délivrabilité

### Impact positif

- **Réputation du domaine** : Protection contre l'usurpation améliore la réputation
- **Confiance des fournisseurs** : Gmail, Yahoo, etc. favorisent les domaines avec DMARC
- **Visibilité** : Les rapports permettent d'identifier et corriger les problèmes

### Précautions

- **Faux positifs** : Une configuration incorrecte peut bloquer des emails légitimes
- **Monitoring continu** : Les rapports doivent être analysés régulièrement
- **Documentation** : Maintenir une documentation des sources d'envoi autorisées

### Exigences des grands fournisseurs

**Gmail et Yahoo (2024)** :

- DMARC obligatoire pour envois en volume (> 5000 emails/jour)
- Minimum `p=none` recommandé
- `p=quarantine` ou `p=reject` fortement conseillé

## Commandes de vérification

### Interroger l'enregistrement DMARC

```bash
# Avec dig
dig TXT _dmarc.exemple.fr +short

# Avec host
host -t TXT _dmarc.exemple.fr

# Avec nslookup
nslookup -type=TXT _dmarc.exemple.fr
```

### Analyser la politique DMARC

```bash
# Script Python simple
python3 << 'EOF'
import dns.resolver

domain = "exemple.fr"
try:
    answers = dns.resolver.resolve(f"_dmarc.{domain}", "TXT")
    for rdata in answers:
        for txt_string in rdata.strings:
            print(txt_string.decode())
except Exception as e:
    print(f"Erreur: {e}")
EOF
```

### Outils en ligne

- **dmarcian.com/dmarc-inspector** : Validation DMARC
- **mxtoolbox.com/dmarc.aspx** : Test complet
- **dmarcanalyzer.com** : Analyse et rapports

## Résumé

**DMARC en bref** :

1. **S'appuie sur SPF et DKIM** pour vérifier l'alignement
2. **Permet de définir une politique** : none, quarantine, reject
3. **Fournit des rapports** sur l'utilisation du domaine
4. **Protège contre l'usurpation** et le phishing
5. **Améliore la délivrabilité** auprès des grands fournisseurs
6. **Nécessite un déploiement progressif** avec monitoring

**Configuration recommandée finale** :

```dns
_dmarc.exemple.fr. IN TXT "v=DMARC1; p=reject; sp=reject; adkim=r; aspf=r; rua=mailto:dmarc-reports@exemple.fr; ruf=mailto:dmarc-forensic@exemple.fr; fo=1; pct=100"
```

DMARC est la pierre angulaire d'une stratégie d'authentification email robuste et complète SPF et DKIM en ajoutant alignement, politique et visibilité.
