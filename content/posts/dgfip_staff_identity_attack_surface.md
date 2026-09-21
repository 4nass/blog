---
author: ["Anass"]
title: "Cyberattaque de la DGFiP : quand l’identité des agents devient la surface d’attaque (in French)"
date: "2026-08-29"
description: "Ce que l’incident DGFiP de 2026 révèle sur les identités internes compromises, l’autorisation, la détection et la réponse aux incidents publics."
tags: ["DGFiP", "Identity Security", "Incident Response", "France", "Privileged Access", "Data Protection"]
categories: ["Identity & Access Systems"]
topics: ["Identity & Access", "Security"]
content_type: "security-analysis"
ShowToc: true
---

La question « Le site des impôts a-t-il été piraté ? » est compréhensible. Elle est toutefois trop simple pour être vraiment utile.

En août 2026, la Direction générale des Finances publiques (DGFiP) a indiqué que des accès illégitimes à une partie de son système d’information avaient permis la consultation et l’extraction de données relatives à des particuliers et à des entreprises. Selon l’explication publique du gouvernement, l’activité observée en juin et juillet faisait suite à l’usurpation d’identifiants d’agents publics. Le chemin technique complet de l’intrusion n’a pas été rendu public.

La même communication établit une distinction importante : le site `impots.gouv.fr` et les espaces en ligne des particuliers et des professionnels n’ont pas été compromis. Les données exposées comprenaient néanmoins des informations fiscales et personnelles sensibles, susceptibles de rendre des campagnes d’hameçonnage, de fraude ou d’usurpation d’identité plus crédibles.

Un autre événement doit également être distingué. Le 17 août, une vulnérabilité technique a été découverte sur le portail des successions vacantes, un service contenant des données publiques. Un acteur malveillant a ensuite revendiqué un accès frauduleux et une extraction de données. Cet événement ne doit pas être fusionné avec l’incident lié aux identités d’agents : les systèmes, la classification des données et la frontière de confiance ne sont pas les mêmes.

Cet article emploie le mot « cyberattaque » dans son titre, tout en conservant le vocabulaire officiel : **accès illégitimes, vol de données et enquête publique en cours**. L’objectif n’est ni de reconstituer une chaîne d’attaque qui n’a pas été publiée, ni d’attribuer l’incident sans éléments. Il s’agit d’examiner ce que ce cas révèle sur l’identité, l’autorisation, l’accès aux données et la réponse aux incidents dans un grand système public.

<!--more-->

## En bref

- **Une identité d’agent peut devenir une surface d’attaque.** L’authentification confirme qu’un identifiant a été accepté ; elle ne prouve ni que l’utilisateur est légitime, ni que chaque action demandée est appropriée.
- **Les portails publics et le système de données interne sont deux frontières différentes.** L’absence de compromission de l’espace d’un contribuable ne signifie pas qu’aucune donnée fiscale n’a pu être exposée ailleurs.
- **L’autorisation doit être plus fine que l’authentification.** Un compte d’agent ne devrait pas pouvoir consulter, rechercher ou exporter toutes les données accessibles à son application.
- **L’accès aux données doit être contrôlé séparément.** Le périmètre, le motif, le volume, l’export et les comportements inhabituels méritent une surveillance dédiée.
- **La MFA est nécessaire, mais pas suffisante.** Une session, un jeton ou une identité déjà approuvée peuvent toujours être utilisés abusivement si les permissions en aval sont trop larges.
- **Les deux événements d’août doivent rester séparés.** L’un concerne l’utilisation abusive d’identités et l’accès à des informations ; l’autre concerne une vulnérabilité technique sur un portail contenant des données publiques.
- **L’ANSSI fait partie d’un dispositif de réponse.** Le CERT-FR apporte des capacités opérationnelles de prévention, de détection et de réponse aux entités publiques prioritaires ; il ne remplace ni le responsable du système, ni l’autorité de protection des données, ni les enquêteurs judiciaires.

> **Périmètre et sécurité :** cet article est une analyse défensive fondée sur les communications publiques. Il n’identifie aucune personne privée, ne teste pas les systèmes de la DGFiP, ne reproduit aucun chemin d’accès et ne fournit ni identifiant ni méthode d’extraction. La démonstration ci-dessous est un modèle d’autorisation en mémoire, sans réseau ni base de données.

## À retenir

| Élément | Détails |
| --- | --- |
| Organisation | Direction générale des Finances publiques (DGFiP) |
| Communication publique | 14 août 2026, avec une page officielle mise à jour le 27 août |
| Période des accès signalés | Juin, juillet et août 2026 ; l’explication liée aux identités d’agents concerne l’activité de juin et juillet |
| Mécanisme indiqué | Usurpation d’identifiants d’agents publics, selon la FAQ gouvernementale |
| Données potentiellement consultées | Identifiant fiscal, état civil, coordonnées, situation fiscale et messages échangés avec la DGFiP |
| Espaces fiscaux publics | `impots.gouv.fr` et espaces en ligne des particuliers et professionnels déclarés non compromis |
| Événement distinct | Vulnérabilité du 17 août sur le portail des successions vacantes, contenant des données publiques |
| Leçon principale | Une identité et une application légitimes peuvent tout de même créer un chemin d’accès non autorisé aux données |
| Angle de l’article | Architecture IAM, autorisation, détection et réponse |

## Ce qui est établi publiquement

Les faits officiels sont plus précis que certains titres de presse.

La DGFiP indique qu’un acteur a revendiqué un vol de données après des accès illégitimes à son système d’information. Sa page publique, modifiée le 27 août, indique que l’activité s’est déroulée en juin, juillet et août 2026 et confirme que le site public ainsi que les espaces en ligne des particuliers et des professionnels n’ont pas été compromis. La FAQ du gouvernement précise que les accès illégitimes de juin et juillet faisaient suite à l’usurpation d’identifiants d’agents publics.

Les catégories de données mentionnées publiquement comprennent notamment :

- les identifiants fiscaux ;
- l’état civil ;
- les coordonnées postales, téléphoniques et électroniques ;
- des éléments de situation fiscale, comme les informations relatives au foyer, le nombre de personnes à charge, le revenu fiscal de référence et le taux de prélèvement à la source ;
- la liste des messages échangés avec la DGFiP via la messagerie en ligne.

Les communications publiques ne donnent pas tous les détails qu’un analyste souhaiterait obtenir : méthode initiale de compromission, application interne concernée, modèle de privilèges, durée des sessions, volume extrait par identité ou chronologie forensique complète. Ces zones d’ombre ne doivent pas être comblées par des suppositions.

Une analyse crédible doit donc distinguer trois niveaux :

1. **Les faits confirmés :** ce que l’administration et l’autorité de protection des données ont déclaré publiquement.
2. **Les mécanismes plausibles :** les manières courantes dont une identité légitime peut être utilisée abusivement.
3. **Les inconnues :** les éléments qui nécessitent des preuves forensiques et ne doivent pas être présentés comme des faits.

Cette distinction est particulièrement importante lorsque l’organisation touchée est une administration publique et que les données concernées sont sensibles.

## Deux événements, deux frontières de confiance

L’incident lié aux identités d’agents et celui du portail des successions vacantes sont apparus à quelques jours d’intervalle, mais ils correspondent à deux problèmes de sécurité différents.

### L’incident lié aux identités d’agents

Le mécanisme indiqué est l’usurpation d’identifiants d’agents publics. La chaîne à examiner concerne donc l’identité des personnels, l’accès aux applications internes, l’autorisation et l’extraction des données.

Le fait qu’un attaquant utilise une identité réelle ou acceptée ne rend pas l’accès légitime. Cela signifie que les contrôles doivent pouvoir distinguer :

- une connexion valide d’un usage valide du compte ;
- une session applicative valide d’une requête de données appropriée ;
- la consultation d’un dossier autorisé d’une collecte anormale ;
- une action utile isolée d’une séquence qui permet une extraction massive.

### Le portail des successions vacantes

La DGFiP a séparément indiqué qu’une vulnérabilité technique avait été découverte le 17 août sur le portail des successions vacantes, qui contient des données publiques. Un acteur malveillant a ensuite revendiqué un accès frauduleux et une extraction de données.

Cette fois, la frontière à analyser est celle d’une application exposée et de ses contrôles de validation des entrées, de session et d’exposition des données. L’événement doit être investigué et corrigé séparément. Le mélanger avec l’incident lié aux identités masquerait les causes et rendrait les mesures correctives moins pertinentes.

La leçon commune pour l’institution n’est pas que tous les systèmes ont échoué de la même façon. C’est qu’une grande administration doit disposer d’un inventaire clair des identités, des applications, des données et des relations de confiance afin de pouvoir isoler les incidents et les corréler lorsque les preuves le justifient.

## Quand l’authentification donne une fausse impression de sécurité

Les systèmes d’identité répondent souvent très bien à la première question : **l’identifiant présenté satisfait-il la politique d’authentification ?**

Les questions plus difficiles viennent ensuite :

- L’utilisateur agit-il depuis un appareil et un contexte attendus ?
- La session reste-t-elle fiable après un événement touchant l’identité ou le jeton ?
- L’application demande-t-elle réellement les données nécessaires à ce rôle ?
- Le volume et la séquence des accès correspondent-ils à l’activité de l’utilisateur ?
- L’utilisateur doit-il voir l’intégralité d’un dossier ou seulement certains champs ?
- Une recherche massive, un export ou l’accès à un historique de messages doit-il exiger une validation supplémentaire ?

Ce sont des questions d’autorisation et de gouvernance des données, pas seulement de connexion.

Un agent peut être autorisé à consulter un dossier dans le cadre d’un traitement. Cela ne signifie pas qu’il doit pouvoir énumérer des millions de dossiers, les exporter en une opération ou accéder à des historiques sans rapport avec son activité. Cette différence doit être imposée par l’application et la couche de données, pas seulement décrite dans une procédure interne.

## Modèle défensif de la chaîne d’accès

La séquence suivante sert à analyser cette catégorie d’incident. Elle ne prétend pas décrire l’implémentation exacte de la DGFiP.

| Étape | Question de sécurité | Contrôle attendu |
| --- | --- | --- |
| Compromission d’identifiant ou de session | Comment un attaquant peut-il agir comme l’agent ? | Authentification résistante à l’hameçonnage, protection des jetons et sécurité du poste |
| Premier accès applicatif | L’identité peut-elle atteindre l’outil depuis ce contexte réseau et cet appareil ? | Assurance de l’appareil, accès conditionnel et segmentation applicative |
| Attribution du rôle | Quelles permissions l’application attribue-t-elle ? | RBAC/ABAC avec moindre privilège et élévation temporaire |
| Consultation d’un dossier | Le dossier est-il dans le périmètre de travail ? | Contraintes de dossier, région, organisation et finalité |
| Champs sensibles | L’utilisateur a-t-il besoin de chaque champ ? | Filtrage au niveau des champs, masquage et minimisation |
| Collecte | Le volume ou la séquence sont-ils anormaux ? | Limites, analyse des requêtes, contrôle des exports et validation renforcée |
| Réponse | L’organisation peut-elle arrêter l’identité et comprendre ses actions ? | Révocation des sessions, rotation des secrets et journaux corrélés |

Si toutes ces décisions sont réduites à « l’utilisateur possède un compte interne », l’identité devient une clé passe-partout. Le périmètre réseau peut rester solide alors que la couche de données demeure trop permissive.

## De l’authentification ponctuelle à la confiance continue

Les informations publiques ne permettent pas d’établir si la MFA était activée, contournée ou absente sur les identités concernées. Il serait donc incorrect d’affirmer que la MFA a échoué dans cet incident précis.

La leçon d’architecture est néanmoins claire : la MFA protège un événement d’authentification, pas toutes les actions effectuées après cet événement. Une session peut être volée. Un jeton peut être rejoué. Un poste compromis peut agir au travers d’un navigateur déjà authentifié. Une identité de support peut enfin être sur-privilégiée alors que son propriétaire s’est correctement authentifié.

Le modèle le plus complet n’est donc pas de demander systématiquement une nouvelle authentification. C’est d’évaluer continuellement la confiance accordée à la session et à l’action en cours. On peut parler de **Continuous Trust Evaluation**, ou d’authentification continue au sens large, à condition de préciser qu’il ne s’agit pas de faire saisir un code à l’utilisateur toutes les cinq minutes.

Une couche d’évaluation continue, branchée sur les solutions IAM existantes, corrèle les signaux qui apparaissent pendant la session :

- changement ou perte de confiance du poste ;
- évolution de la localisation ou du réseau ;
- réutilisation inhabituelle d’un jeton ou d’une session ;
- comportement anormal dans l’application ;
- volume ou séquence de consultation inhabituels ;
- événement métier sensible, comme un export ou une modification de privilège ;
- signal provenant de l’EDR, du SIEM, du VPN, du PAM ou d’un autre fournisseur d’identité.

Un moteur de risque peut ensuite prendre une décision graduée et explicable :

| Niveau de confiance | Réponse possible |
| --- | --- |
| Confiance normale | Continuer l’accès sans friction supplémentaire |
| Risque modéré | Renforcer la journalisation, limiter le débit ou demander une confirmation légère |
| Risque élevé sur une action sensible | Demander une authentification renforcée uniquement pour cette action |
| Confiance fortement dégradée | Révoquer la session, bloquer l’action et alerter le SOC |

Le step-up reste donc utile, mais comme une réponse conditionnelle dans une chaîne de décision, pas comme une friction appliquée uniformément à tous les utilisateurs. Cette approche protège mieux le confort utilisateur : une session qui reste cohérente ne doit pas être interrompue, tandis qu’un export inhabituel ou un changement de contexte peut déclencher une mesure proportionnée.

Pour être exploitable par un RSSI, un DPO ou un auditeur, chaque décision doit rester lisible : signaux observés, facteurs de risque, seuil appliqué, action choisie et possibilité de retour arrière. Le moteur ne doit pas être une boîte noire qui remplace l’IAM ; il doit devenir une couche transverse de décision, connectée aux systèmes d’identité et aux événements métier.

Cette évaluation continue doit également respecter la minimisation des données. Il n’est pas nécessaire de centraliser tous les détails comportementaux d’un utilisateur pour détecter un export anormal. Les signaux utiles, leur durée de conservation, leur finalité et les personnes autorisées à les consulter doivent être documentés.

## Contrôles pour une plateforme publique de données

### Identités des agents

Maintenir un inventaire complet des identités humaines, techniques et d’urgence. Supprimer rapidement les comptes inactifs, séparer les comptes administratifs des comptes ordinaires et limiter dans le temps les accès privilégiés. Lorsqu’un agent change de fonction, d’organisation ou d’affectation, il ne devrait pas conserver automatiquement son ancien périmètre d’accès.

### Autorisation applicative

Ne pas s’arrêter à des rôles larges comme « agent », « responsable » ou « administrateur ». Ajouter des contraintes correspondant au travail réel : dossier, région, organisation, type de données, finalité et période. Les imposer dans l’application et l’API, pas uniquement dans l’interface utilisateur.

### Données et exports

Traiter l’export massif comme une capacité distincte. Exiger une justification, une limite de volume, une validation supplémentaire ou une authentification renforcée selon le contexte. Appliquer un masquage au niveau des champs et protéger davantage l’historique des messages, les documents d’identité et les informations financières que les métadonnées ordinaires d’un dossier.

### Appareils et sessions

Utiliser des signaux liés aux appareils gérés et à l’accès conditionnel pour les applications internes, sans considérer l’emplacement réseau comme une preuve de légitimité. Révoquer les sessions après un événement d’identité à haut niveau de confiance et s’assurer que cette révocation atteint toutes les applications qui acceptent le jeton ou la session.

### Identités techniques

Séparer les accès humains des accès entre applications. Un compte de service utilisé par un workflow ne doit pas hériter des permissions interactives d’un agent. Faire tourner les secrets, limiter les audiences et journaliser chaque opération sensible réalisée par le service.

### Observabilité de la couche de données

L’application devrait journaliser qui a consulté quoi, par quelle fonction, dans quel périmètre et avec quel résultat. Les journaux de base de données, d’API et d’identité doivent partager un identifiant de corrélation afin de permettre la reconstitution d’une séquence plutôt que l’examen d’événements isolés.

## Détection et investigation

Les échecs de connexion ne sont pas les seuls signaux utiles. Dans cette catégorie d’incident, l’activité suspecte peut être constituée de requêtes réussies effectuées via une application légitime.

Les détections pertinentes comprennent notamment :

- une identité d’agent qui accède à des applications hors de son affectation habituelle ;
- un accès depuis un nouvel appareil, une nouvelle région ou un nouveau réseau suivi de requêtes sensibles ;
- le passage soudain de consultations unitaires à des accès séquentiels ou volumineux ;
- des consultations répétées d’historiques de messages ou de données sans rapport avec l’activité de l’agent ;
- une identité technique qui réalise des actions interactives ;
- un export qui survient juste avant un changement de session, de rôle ou d’appareil ;
- plusieurs identités qui partagent le même appareil, la même source ou la même séquence inhabituelle ;
- des journaux qui enregistrent l’authentification mais pas les dossiers, champs ou volumes réellement consultés.

L’investigation doit corréler au moins cinq chronologies :

1. les événements du fournisseur d’identité et l’émission des jetons ;
2. les événements du poste, du navigateur et de l’assurance de l’appareil ;
3. les requêtes applicatives et les décisions d’autorisation ;
4. les requêtes de base de données, exports et accès aux objets ;
5. les activités réseau, proxy et administration.

La question clé n’est pas seulement « quel compte a été utilisé ? ». Elle est : « que pouvait faire cette identité, que fit-elle réellement et quel contrôle aurait dû arrêter ou signaler l’action ? »

## Le rôle de l’ANSSI et des autres acteurs

La communication publique résume parfois la réponse sous le nom d’une seule institution. En pratique, la réponse est distribuée.

### La DGFiP et le ministère des Finances

Le responsable du système doit contenir l’incident, préserver les preuves, évaluer les données touchées, corriger le chemin d’accès et informer les personnes concernées. Il doit également vérifier que les mesures prises n’ont pas créé un second problème de disponibilité ou d’accès.

### L’ANSSI et le CERT-FR

Le CERT-FR est l’entité opérationnelle par laquelle l’ANSSI met en œuvre la mission d’autorité de défense des systèmes numériques d’intérêt pour la Nation. Son périmètre publié couvre la prévention, la détection, la réponse et la coordination des vulnérabilités, avec un soutien prioritaire aux ministères, services de l’État et autres entités désignées. Dans le cas de la DGFiP, le ministère des Finances a annoncé une conférence de suivi en présence du directeur général adjoint de l’ANSSI.

Ce rôle doit être compris avec précision. L’ANSSI peut apporter son expertise, sa coordination, son soutien de crise et sa vision nationale de la menace. Elle ne remplace pas les responsabilités propres de l’organisation en matière d’IAM, de sécurité, de protection des données et d’exploitation.

### La CNIL

La CNIL a indiqué avoir été informée des violations de données affectant le système d’information de la DGFiP et poursuivre ses vérifications. Il s’agit d’une dimension réglementaire et de protection des données, distincte du confinement technique et d’une éventuelle enquête pénale.

### Les autorités judiciaires et services d’enquête

L’attribution, la collecte des preuves et les poursuites répondent à leurs propres règles. Un article public ne doit pas désigner un acteur ou rattacher l’incident à une campagne géopolitique sans attribution officielle ou éléments techniques solides.

La leçon pratique est qu’une réponse de crise a besoin d’une chronologie partagée et de responsabilités claires, pas d’une étiquette « cyber » unique recouvrant tout.

## Réponse à un incident lié à une identité

1. **Contenir l’identité.** Désactiver ou restreindre les comptes suspects, révoquer les sessions et invalider les jetons de service selon les éléments disponibles.
2. **Préserver les preuves avant le nettoyage massif.** Protéger les journaux d’identité, de poste, d’application, de base de données, d’export et de réseau.
3. **Cartographier les permissions effectives.** Documenter ce que chaque identité pouvait réellement consulter, et pas uniquement le rôle affiché dans l’annuaire.
4. **Délimiter les données.** Déterminer les dossiers, champs et messages effectivement consultés ou extraits, en séparant l’accès confirmé de l’accès théorique.
5. **Faire tourner les secrets accessibles.** Considérer les jetons, clés et identifiants exposés par le chemin concerné comme compromis jusqu’à leur examen.
6. **Rechercher les réutilisations.** Chercher la même identité, le même appareil, la même audience de jeton, la même source et la même séquence dans les applications connectées.
7. **Protéger les personnes concernées.** Communiquer clairement les données impliquées et les formes de fraude susceptibles de suivre.
8. **Réparer la frontière.** Réduire les privilèges, ajouter des contrôles de requêtes et d’exports, corriger la vulnérabilité technique éventuelle et tester la réponse dans un environnement isolé.

La huitième étape est la véritable mesure du succès. Réinitialiser les mots de passe sans réduire le périmètre de permissions laisse la même surface d’attaque prête pour la prochaine identité compromise.

## Ce que les institutions publiques doivent retenir

La leçon principale n’est pas que les utilisateurs internes seraient indignes de confiance. C’est que la confiance doit être précise et réévaluée en continu.

Une administration peut disposer d’une bonne sécurité périmétrique, d’une connexion publique correctement protégée et d’un annuaire bien administré, tout en exposant trop de données via une application interne. Le plan de contrôle et le plan de données doivent être cohérents :

- l’identité doit être suffisamment fiable pour l’action demandée ;
- l’appareil et la session doivent être suffisamment fiables pour le contexte ;
- l’application doit être autorisée à accéder à la ressource demandée ;
- la requête doit être proportionnée à la finalité ;
- le résultat doit être observable et révocable.

Il s’agit à la fois d’un problème IAM, d’un problème de gouvernance des données et d’un problème de réponse aux incidents.

## Conclusion

L’incident DGFiP de 2026 rappelle qu’une compromission ne doit pas nécessairement commencer sur la page de connexion publique. Un attaquant peut obtenir un impact en abusant d’une identité déjà reconnue par une application interne, surtout lorsque l’authentification, l’autorisation et l’accès aux données sont traités comme une seule décision.

Le fait que les espaces fiscaux publics n’aient pas été compromis est important. Il réduit le périmètre ; il n’efface pas le risque créé par l’exposition de données fiscales et personnelles ailleurs dans le système d’information. L’événement distinct du portail des successions vacantes rappelle également que des titres similaires peuvent recouvrir des vulnérabilités différentes et nécessiter des réponses différentes.

Le principe durable est simple : **une identité d’agent ne devrait ouvrir que la plus petite porte nécessaire à la tâche en cours, et toute action sensible devrait laisser une trace que l’organisation peut examiner et révoquer**.

## Sources

1. [DGFiP — Vol de données suite à des accès illégitimes au système d’information](https://www.impots.gouv.fr/actualite/vol-de-donnees-suite-des-acces-illegitimes-au-systeme-dinformation-de-la-dgfip), page officielle publiée le 14 août et modifiée le 27 août 2026.
2. [Ministère des Finances — FAQ sur les accès illégitimes au système d’information de la DGFiP](https://www.economie.gouv.fr/actualites/acces-illegitimes-au-systeme-dinformation-de-la-direction-generale-des-finances-publiques-consulter-la-faq), publiée le 21 août 2026.
3. [CNIL — Piratage du système d’information des impôts](https://www.cnil.fr/fr/piratage-du-systeme-dinformation-des-impots-les-verifications-sont-en-cours), notification et vérifications en cours, 18 août 2026.
4. [Ministère des Finances — Conférence de suivi de la réponse à la cyberattaque visant la DGFiP](https://presse.economie.gouv.fr/ip-conference-de-presse-sur-le-suivi-de-la-reponse-a-la-cyberattaque-visant-la-direction-generale-des-finances-publiques-mardi-18-aout-2026/), participation de l’ANSSI, 18 août 2026.
5. [ANSSI — CERT-FR](https://cyber.gouv.fr/nous-connaitre/lagence/organisation/cert-fr/), missions opérationnelles de prévention, détection et réponse.
6. [NIST SP 800-207 — Zero Trust Architecture](https://csrc.nist.gov/pubs/sp/800/207/final), vérification continue et accès fondé sur les ressources.
7. [NIST SP 800-53 — Security and Privacy Controls](https://csrc.nist.gov/pubs/sp/800/53/r5/upd1/final), contrôle d’accès, audit et réponse aux incidents.
