---
title: "SRS - forwarding email"
---
# SRS (Sender Rewriting Scheme)

## Introduction

Le _Sender Rewriting Scheme_ (SRS) est une technique utilisée par les serveurs de transfert de courrier (_forwarders_) pour réécrire l’adresse d’expéditeur de l’enveloppe SMTP.  
Son objectif principal est de préserver la compatibilité avec SPF lorsqu’un message est transféré vers un autre serveur.

SRS est défini dans la [RFC 8617](https://www.rfc-editor.org/rfc/rfc8617.html), qui documente notamment l’usage de SRS dans le contexte du transfert de courrier.  
Il ne s’agit ni d’un mécanisme d’authentification comparable à DKIM ou DMARC, ni d’une garantie que le contenu du message est légitime : c’est une transformation de l’adresse d’enveloppe, à appliquer par le système qui effectue le transfert.

## Le problème du SPF lors d’un transfert

SPF évalue si l’adresse IP qui se connecte au serveur destinataire est autorisée à émettre pour le domaine de l’expéditeur de l’enveloppe SMTP (`MAIL FROM`).  
Lors d’un transfert, l’IP visible par le destinataire est celle du serveur de transfert, et non celle du serveur d’origine. Si le domaine d’origine n’autorise pas cette IP dans son enregistrement SPF, le contrôle peut échouer.

Par exemple, Alice envoie un message depuis `alice@example.org` vers une boîte qui redirige le courrier vers `bob@destination.net` :

1. le serveur d’origine envoie le message au serveur qui héberge la redirection ;
2. ce serveur le retransmet vers `destination.net` ;
3. `destination.net` voit une connexion venant de l’IP du serveur de transfert ;
4. SPF vérifie toujours le domaine d’origine de l’enveloppe, `example.org`, qui n’a aucune raison d’autoriser le serveur de transfert.

Le résultat SPF peut donc devenir `fail` ou `softfail`. Ce phénomène est souvent appelé _forwarding problem_. Il peut contribuer à l’échec de DMARC si le message ne dispose pas d’un autre mécanisme aligné qui passe, par exemple une signature DKIM encore valide.

!!! note
SPF vérifie l’expéditeur de l’enveloppe SMTP, pas nécessairement l’adresse visible dans l’en-tête `From:`. SRS modifie l’enveloppe et ne doit pas être confondu avec la réécriture de cet en-tête.

## Principe de fonctionnement

Avant de relayer le message, le serveur de transfert remplace l’adresse d’enveloppe par une adresse de son propre domaine. Le destinataire peut alors vérifier SPF sur le domaine du forwarder, dont l’enregistrement SPF autorise son serveur d’envoi.

Une adresse SRS contient typiquement :

- un préfixe indiquant le schéma, comme `SRS0` ou `SRS1` ;
- une information temporelle permettant une expiration ;
- une empreinte cryptographique calculée avec un secret local ;
- l’adresse ou le domaine d’origine, encodé dans la partie locale.

Exemple illustratif :

```text
SRS0=HHH=TT=example.org=alice@forwarder.example
```

Cette adresse n’est qu’un exemple : le format exact et les séparateurs varient selon l’implémentation. L’empreinte rend plus difficile la fabrication d’adresses SRS arbitraires et permet au serveur de vérifier qu’il a lui-même produit l’adresse avant d’en extraire l’adresse d’origine.

### SRS0

Lors du transfert initial, l’adresse d’enveloppe d’origine est réécrite en une adresse SRS sur le domaine du serveur de transfert. Celui-ci doit être autorisé à envoyer pour ce domaine dans son SPF.

```text
Avant : MAIL FROM:<alice@example.org>
Après : MAIL FROM:<SRS0=...@forwarder.example>
```

Le destinataire voit une connexion du forwarder et évalue SPF pour `forwarder.example`, plutôt que pour `example.org`.

### SRS1 et transferts en chaîne

Un message peut être transféré plusieurs fois. Le second forwarder ne peut pas simplement conserver l’adresse SRS du premier : il doit à son tour utiliser une adresse de son propre domaine afin que son SPF soit vérifiable. SRS1 permet d’encoder l’adresse SRS précédente dans une nouvelle adresse.

```text
Adresse SRS du premier forwarder
        ↓ nouvelle réécriture
Adresse SRS1 sur le domaine du second forwarder
```

À la réception d’un message de retour, les forwarders successifs valident leur propre empreinte et retirent une couche de réécriture. La chaîne peut ainsi remonter jusqu’à l’adresse d’enveloppe d’origine. Les détails de l’encodage SRS1 dépendent de l’implémentation ; il faut vérifier qu’elle prend en charge le nombre de sauts attendu.

## Gestion des retours et des rebonds

L’adresse d’enveloppe sert aussi à acheminer les notifications de non-remise (_bounces_). Si un message transféré ne peut pas être livré, le serveur destinataire renvoie normalement le rapport à l’adresse SRS. Le forwarder reçoit ce retour, vérifie l’adresse réécrite, restaure l’adresse d’enveloppe précédente, puis relaie le rapport vers l’expéditeur d’origine.

Le domaine qui émet les adresses SRS doit donc :

- accepter les messages adressés à ces adresses, généralement par une règle de réécriture ou un domaine virtuel ;
- pouvoir reconnaître les adresses générées et les vérifier ;
- disposer d’un traitement fiable des retours et des délais de rétention ;
- limiter les destinataires et le débit afin de prévenir les abus et les boucles de rebonds.

Une adresse SRS n’est pas une adresse permanente à communiquer aux utilisateurs. Elle est une adresse technique de retour pour un message transféré.

## Ce que SRS apporte — et ce qu’il n’apporte pas

### Apports

- **Réduit les échecs SPF liés au transfert.** Le SPF est évalué sur le domaine du forwarder, qui peut autoriser ses propres serveurs.
- **Préserve le chemin des erreurs de livraison.** Les bounces reçus à l’adresse réécrite peuvent être renvoyés à l’expéditeur d’origine.
- **Rend le transfert plus prévisible.** Les destinataires n’ont pas à autoriser individuellement l’IP de chaque forwarder dans le SPF du domaine d’origine.

### Limites

- **Ne préserve pas à lui seul DMARC.** DMARC exige qu’au moins SPF ou DKIM passe avec alignement sur le domaine du `From:`. Après réécriture, SPF peut passer pour le forwarder, mais son domaine n’est généralement pas aligné avec le domaine d’origine. Une signature DKIM valide et alignée peut continuer à faire passer DMARC ; si elle est absente ou invalidée par une modification du message, DMARC peut échouer.
- **Ne répare pas DKIM.** Le changement de l’enveloppe n’altère normalement pas les en-têtes ni le corps signés, mais une modification du contenu lors du transfert peut invalider la signature.
- **Ne fournit pas de preuve d’authenticité du message d’origine.** Un SPF réussi pour le domaine du forwarder atteste seulement que celui-ci est autorisé à envoyer pour son propre domaine.
- **N’empêche pas les boucles ni les abus à lui seul.** Le serveur doit valider les empreintes, appliquer une expiration, limiter les réécritures et empêcher les boucles de transfert.
- **N’est pas toujours reconnu par les systèmes destinataires.** Certains filtres peuvent traiter avec suspicion les adresses d’expéditeur réécrites, notamment si le domaine SRS n’est pas correctement configuré.

## Relation avec SPF, DKIM, DMARC et ARC

| Mécanisme | Rôle dans le transfert                                                                                                                                                    |
| --------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| SPF       | SRS permet au SPF de contrôler le domaine du serveur qui retransmet le message.                                                                                           |
| DKIM      | Peut conserver une authentification alignée avec le domaine du `From:` si la signature reste valide.                                                                      |
| DMARC     | Peut encore échouer si SPF n’est plus aligné et qu’aucune signature DKIM alignée ne passe.                                                                                |
| ARC       | Peut transmettre des résultats d’authentification observés par les intermédiaires ; il ne remplace pas SRS et le destinataire décide s’il fait confiance à la chaîne ARC. |
| SRS       | Réécrit l’expéditeur d’enveloppe pour traiter le problème SPF du relais et permettre le retour des bounces.                                                               |

SRS et ARC sont donc complémentaires. SRS adapte l’enveloppe pour que SPF puisse être évalué sur le serveur de transfert ; ARC peut conserver le contexte des contrôles effectués avant qu’un transfert ou une modification ne fasse échouer l’authentification au destinataire final.

## Configuration et précautions opérationnelles

SRS doit être activé sur le système qui retransmet effectivement les messages, avant leur sortie vers le serveur suivant. Une configuration typique consiste à installer une fonction ou un service SRS, puis à l’intégrer aux règles de réécriture de l’enveloppe du MTA. Les paramètres précis dépendent du MTA et du logiciel choisi ; il n’existe pas de configuration universelle à copier telle quelle.

Points de contrôle importants :

1. **Choisir le domaine SRS.** Utiliser un domaine ou sous-domaine maîtrisé par l’organisation, par exemple `forwarder.example`, et publier un SPF autorisant les IP de sortie réelles.
2. **Protéger le secret.** L’empreinte doit être calculée avec un secret aléatoire, stocké hors des fichiers accessibles publiquement et sauvegardé de manière contrôlée. Une rotation mal planifiée peut empêcher de reconnaître les anciennes adresses SRS encore en circulation ; certaines implémentations acceptent temporairement plusieurs secrets.
3. **Définir l’expiration.** L’information temporelle doit empêcher la réutilisation indéfinie des adresses, tout en laissant un délai raisonnable pour les bounces tardifs.
4. **Traiter les adresses longues.** L’encodage peut dépasser les limites de longueur d’une adresse ou de sa partie locale lorsque l’adresse d’origine est longue. Vérifier le comportement de l’implémentation et éviter les troncatures.
5. **Ne réécrire que ce qui doit l’être.** Appliquer SRS aux messages relayés, pas nécessairement aux messages sortants normaux du domaine. Traiter avec prudence les expéditeurs vides (`MAIL FROM:<>`) utilisés par les bounces : les réécrire peut créer des boucles.
6. **Éviter les boucles.** Reconnaître les adresses SRS déjà produites, valider leur empreinte et appliquer des limites de profondeur et de débit.
7. **Préserver les traces.** Conserver des journaux suffisants pour diagnostiquer les échecs de validation, les adresses expirées et les bounces non distribués, sans exposer le secret.
8. **Contrôler les règles d’acheminement.** Les messages reçus à une adresse SRS doivent aboutir au service qui peut la vérifier et restaurer l’adresse précédente, et non être traités comme du courrier ordinaire.

!!! warning
    Ne restaurez pas une adresse d’origine à partir d’une chaîne SRS non vérifiée.  
    Une restauration sans validation cryptographique permettrait à un tiers de faire envoyer des bounces vers une adresse choisie par lui, et pourrait transformer le serveur en relais d’abus.

## Vérification

Après activation, effectuer un test de bout en bout avec une boîte qui transfère vers un serveur extérieur que l’on contrôle :

1. envoyer un message depuis un domaine de test vers la boîte transférante ;
2. vérifier dans les journaux SMTP sortants que `MAIL FROM` a été remplacé par une adresse SRS sur le domaine attendu ;
3. examiner les en-têtes `Authentication-Results` du destinataire final : le SPF devrait porter sur le domaine SRS et passer si le DNS et l’IP de sortie sont correctement configurés ;
4. vérifier séparément DKIM et DMARC, en tenant compte de l’alignement : un SPF passant pour le domaine SRS ne signifie pas que DMARC passe ;
5. provoquer un échec de livraison contrôlé et vérifier que le bounce revient au forwarder puis est acheminé vers l’expéditeur d’origine ;
6. tester une adresse SRS altérée ou expirée, une adresse d’origine longue et un rebond (`MAIL FROM:<>`) pour confirmer que le traitement refuse ou gère ces cas comme prévu ;
7. si plusieurs forwarders sont impliqués, vérifier la réécriture et la restauration à chaque niveau (SRS1 compris).

Ne validez pas seulement le fait qu’un message arrive dans la boîte finale : il faut également contrôler l’expéditeur d’enveloppe observé, les résultats d’authentification et le trajet d’un bounce.

## Conclusion

SRS traite un problème précis : le SPF d’un message qui quitte un serveur différent de celui autorisé par le domaine d’origine, comme lors d’un forwarding. Le forwarder réécrit l’expéditeur d’enveloppe sous son propre domaine, puis utilise cette adresse pour recevoir et réacheminer les bounces. Cela améliore le fonctionnement du transfert, mais ne garantit pas le succès de DMARC et ne remplace ni DKIM ni ARC. Une mise en œuvre sûre exige une gestion correcte du secret, de l’expiration, des retours, des rebonds et des chaînes de transfert.

## Références

- [RFC 8617 — The Authenticated Received Chain (ARC) Protocol](https://www.rfc-editor.org/rfc/rfc8617.html)
- [RFC 7208 — Sender Policy Framework (SPF)](https://www.rfc-editor.org/rfc/rfc7208.html)
- [RFC 7489 — Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://www.rfc-editor.org/rfc/rfc7489.html)
- [RFC 6376 — DomainKeys Identified Mail (DKIM) Signatures](https://www.rfc-editor.org/rfc/rfc6376.html)
