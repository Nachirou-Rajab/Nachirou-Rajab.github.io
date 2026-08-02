# Cœur de réseau redondant à routage dynamique

> **Contexte** : laboratoire personnel — Licence Professionnelle ASUR, INPTIC
> **Plateforme** : PNETLab / Cisco IOS
> **Période** : 2026

[← Retour au portfolio](../index.md)

---

## 1. Résumé exécutif

[À COMPLÉTER — 2 à 3 phrases : quel besoin ce lab simule-t-il ? Par exemple, un cœur de réseau
d'entreprise multi-sites devant continuer à fonctionner malgré la panne d'un lien ou d'un
équipement de distribution.]

---

## 2. Cahier des charges

| Exigence | Traduction technique |
|---|---|
| Routage entre plusieurs sites | OSPF multi-aires (backbone + aires périphériques) |
| Sécuriser les échanges de routage | Authentification des adjacences |
| Limiter la taille des tables de routage | Résumation aux ABR, aire stub |
| Continuité de service passerelle | HSRP avec priorité et préemption |
| Continuité et débit sur les liens | EtherChannel LACP |
| Convergence rapide en couche 2 | Rapid-PVST, root bridge maîtrisé |

### Plan d'adressage

[À COMPLÉTER : tableau des aires OSPF, des réseaux et des interfaces.]

---

## 3. Schéma d'architecture

![Topologie du cœur de réseau redondant](../assets/images/topologie-coeur-redondant.png)

[À COMPLÉTER : exporte ta topologie PNETLab ou refais-la sous Draw.io, avec les aires OSPF,
les adresses et les noms d'interfaces.]

---

## 4. Procédure de déploiement

### 4.1 Routage OSPF multi-aires

[À COMPLÉTER : configuration des processus OSPF, déclaration des aires, authentification,
résumation aux ABR, redistribution de la route par défaut.]

### 4.2 Haute disponibilité HSRP

[À COMPLÉTER : groupes HSRP par VLAN, priorités, préemption, adresse virtuelle.]

### 4.3 Agrégation LACP

[À COMPLÉTER : configuration du port-channel entre commutateurs.]

### 4.4 Optimisation du spanning-tree

[À COMPLÉTER : mode Rapid-PVST, élection forcée du root bridge, PortFast, BPDU Guard.]

---

## 5. Recette et tests

| # | Test | Méthode | Résultat attendu | Statut |
|---|---|---|---|---|
| 1 | Adjacences OSPF | `show ip ospf neighbor` | Voisins en état FULL | ☐ |
| 2 | Table de routage | `show ip route ospf` | Routes inter-aires présentes | ☐ |
| 3 | Résumation | `show ip route` sur un routeur d'aire | Route résumée unique | ☐ |
| 4 | État HSRP | `show standby brief` | Rôles actif/veille conformes | ☐ |
| 5 | Agrégation | `show etherchannel summary` | Port-channel en état actif | ☐ |
| 6 | Spanning-tree | `show spanning-tree` | Root bridge conforme au plan | ☐ |
| 7 | **Bascule** | Ping continu + coupure du lien primaire | Perte minimale, reprise automatique | ☐ |

**Le test 7 est le plus important** : c'est lui qui prouve que la redondance fonctionne
réellement. Lance un `ping -t` depuis un poste, coupe le lien ou éteins l'équipement actif,
et capture le nombre exact de paquets perdus avant reprise. Note ce chiffre — c'est un
résultat quantifié, exactement ce qu'un recruteur cherche.

---

## 6. Difficultés rencontrées et enseignements

### Difficulté 1 — [À COMPLÉTER]

**Symptôme observé :** …
**Diagnostic :** …
**Résolution :** …
**Ce que j'en retiens :** …

---

[← Retour au portfolio](../index.md)
