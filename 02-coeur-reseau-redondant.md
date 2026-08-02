# Cœur de réseau redondant à haute disponibilité

> **Contexte** : laboratoire personnel — Licence Professionnelle ASUR, INPTIC
> **Plateforme** : PNETLab (routeurs et commutateurs Cisco IOS)
> **Période** : 2026

[← Retour au portfolio](index.md)

---

## 1. Résumé exécutif

Dans un réseau d'entreprise, la panne d'une seule passerelle suffit à couper l'accès de tout un
service. Ce laboratoire répond à une question précise : comment concevoir une infrastructure qui
continue de fonctionner malgré la perte d'un routeur ou d'un lien, sans intervention humaine et
sans reconfiguration des postes clients ?

L'architecture repose sur deux routeurs assurant conjointement le rôle de passerelle pour deux
VLAN, et sur deux commutateurs reliés par une agrégation de liens. Chaque routeur est passerelle
active pour un VLAN et passerelle de secours pour l'autre : les deux équipements travaillent en
permanence plutôt que d'en laisser un inactif, et la défaillance de l'un fait basculer
automatiquement ses VLAN vers l'autre.

---

## 2. Cahier des charges

| Exigence | Traduction technique |
|---|---|
| Aucun point de défaillance unique sur la passerelle | HSRP v2 avec adresse virtuelle et préemption |
| Répartir la charge entre les deux routeurs | Un groupe HSRP par VLAN, rôles inversés |
| Redondance et débit entre commutateurs | EtherChannel LACP sur deux liens |
| Routage inter-VLAN sur un lien unique | Sous-interfaces 802.1Q (router-on-a-stick) |
| Convergence rapide de la couche 2 | Rapid-PVST, root bridge maîtrisé |
| Sécurisation de la couche d'accès | VLAN natif dédié, PortFast et BPDU Guard |

### Plan d'adressage

| VLAN | Nom | Réseau | Passerelle virtuelle | Poste |
|---|---|---|---|---|
| 10 | UTILISATEURS | 192.168.10.0/24 | 192.168.10.254 | VPC7 — 192.168.10.11 |
| 20 | SERVEURS | 192.168.20.0/24 | 192.168.20.254 | VPC8 — 192.168.20.11 |
| 99 | NATIF-PARKING | — | — | VLAN natif des trunks |

| Équipement | VLAN 10 | VLAN 20 | Liaison inter-routeurs |
|---|---|---|---|
| R4 | 192.168.10.252 | 192.168.20.252 | 10.10.10.1/30 |
| R3 | 192.168.10.253 | 192.168.20.253 | 10.10.10.2/30 |

### Répartition des rôles

| VLAN | HSRP actif | HSRP secours | Root bridge STP |
|---|---|---|---|
| 10 | R4 (priorité 110) | R3 | Switch5 |
| 20 | R3 (priorité 110) | R4 | Switch6 |

> **Choix de conception.** Plutôt que de désigner un routeur maître pour tout le réseau, chaque
> VLAN se voit attribuer une passerelle active différente. Les deux routeurs acheminent donc du
> trafic en permanence, ce qui évite d'immobiliser un équipement coûteux en simple attente. Le
> root bridge du spanning-tree est aligné sur le commutateur raccordé au routeur actif, afin que
> le chemin de couche 2 ne traverse pas inutilement l'agrégation inter-commutateurs.

---

## 3. Schéma d'architecture

![Topologie du cœur de réseau redondant](topologie-coeur-redondant.png)

```
     R4 ──────── Gi0/2 ══ Gi0/0 ──────── R3
   HSRP actif V10                   HSRP actif V20
      │ Gi0/0                        Gi0/1 │
      │                                    │
   Gi0/0                                Gi0/2
  [ Switch5 ] ══ Po1 — LACP (x2) ══ [ Switch6 ]
      │ Gi0/1                        Gi0/1 │
     VPC7                                VPC8
   VLAN 10                              VLAN 20
```

Chaque poste atteint sa passerelle par le chemin direct en fonctionnement nominal. En cas de
panne du routeur actif, le trafic emprunte l'agrégation entre commutateurs pour rejoindre le
routeur de secours.

---

## 4. Procédure de déploiement

### 4.1 Configuration de R4 — passerelle active du VLAN 10

```
hostname R4
!
! --- Liaison trunk vers Switch5 (router-on-a-stick) ---
interface GigabitEthernet0/0
 description Trunk 802.1Q vers Switch5
 no ip address
 no shutdown
!
interface GigabitEthernet0/0.10
 description VLAN Utilisateurs - HSRP actif
 encapsulation dot1Q 10
 ip address 192.168.10.252 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 priority 110
 standby 10 preempt
 standby 10 timers 1 3
!
interface GigabitEthernet0/0.20
 description VLAN Serveurs - HSRP secours
 encapsulation dot1Q 20
 ip address 192.168.20.252 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 preempt
 standby 20 timers 1 3
!
interface GigabitEthernet0/0.99
 description VLAN natif
 encapsulation dot1Q 99 native
!
! --- Liaison directe vers R3 ---
interface GigabitEthernet0/2
 description Liaison inter-routeurs vers R3
 ip address 10.10.10.1 255.255.255.252
 no shutdown
!
! --- Durcissement de base ---
no ip domain-lookup
service password-encryption
```

> **Choix de conception.** La préemption est activée sur les deux groupes. Sans elle, un routeur
> revenu en service après une panne resterait indéfiniment en secours : la répartition de charge
> serait perdue et les deux VLAN se retrouveraient portés par le même équipement. Les
> temporisateurs sont abaissés à 1 seconde pour les messages hello et 3 secondes pour le délai
> de bascule, contre 3 et 10 par défaut, ce qui réduit la coupure perçue par l'utilisateur.

### 4.2 Configuration de R3 — passerelle active du VLAN 20

Configuration miroir : les priorités sont inversées.

```
hostname R3
!
interface GigabitEthernet0/1
 description Trunk 802.1Q vers Switch6
 no ip address
 no shutdown
!
interface GigabitEthernet0/1.10
 description VLAN Utilisateurs - HSRP secours
 encapsulation dot1Q 10
 ip address 192.168.10.253 255.255.255.0
 standby version 2
 standby 10 ip 192.168.10.254
 standby 10 preempt
 standby 10 timers 1 3
!
interface GigabitEthernet0/1.20
 description VLAN Serveurs - HSRP actif
 encapsulation dot1Q 20
 ip address 192.168.20.253 255.255.255.0
 standby version 2
 standby 20 ip 192.168.20.254
 standby 20 priority 110
 standby 20 preempt
 standby 20 timers 1 3
!
interface GigabitEthernet0/1.99
 description VLAN natif
 encapsulation dot1Q 99 native
!
interface GigabitEthernet0/0
 description Liaison inter-routeurs vers R4
 ip address 10.10.10.2 255.255.255.252
 no shutdown
!
no ip domain-lookup
service password-encryption
```

### 4.3 Configuration de Switch5

```
hostname Switch5
!
vlan 10
 name UTILISATEURS
vlan 20
 name SERVEURS
vlan 99
 name NATIF-PARKING
!
spanning-tree mode rapid-pvst
spanning-tree vlan 10 priority 4096
spanning-tree vlan 20 priority 8192
spanning-tree portfast bpduguard default
!
! --- Trunk vers le routeur R4 ---
interface GigabitEthernet0/0
 description Trunk vers R4
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 no shutdown
!
! --- Agregation LACP vers Switch6 ---
interface range GigabitEthernet0/2 - 3
 description Agregation LACP vers Switch6
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description Lien inter-commutateurs agrege
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
!
! --- Port utilisateur ---
interface GigabitEthernet0/1
 description Poste VPC7
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
 no shutdown
!
! --- Ports inutilises ---
interface range GigabitEthernet1/0 - 3
 switchport mode access
 switchport access vlan 99
 shutdown
```

### 4.4 Configuration de Switch6

```
hostname Switch6
!
vlan 10
 name UTILISATEURS
vlan 20
 name SERVEURS
vlan 99
 name NATIF-PARKING
!
spanning-tree mode rapid-pvst
spanning-tree vlan 20 priority 4096
spanning-tree vlan 10 priority 8192
spanning-tree portfast bpduguard default
!
! --- Trunk vers le routeur R3 ---
interface GigabitEthernet0/2
 description Trunk vers R3
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 no shutdown
!
! --- Agregation LACP vers Switch5 ---
interface GigabitEthernet0/0
 description Agregation LACP vers Switch5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 channel-group 1 mode active
 no shutdown
!
interface GigabitEthernet0/3
 description Agregation LACP vers Switch5
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
 channel-group 1 mode active
 no shutdown
!
interface Port-channel1
 description Lien inter-commutateurs agrege
 switchport trunk encapsulation dot1q
 switchport mode trunk
 switchport trunk native vlan 99
 switchport trunk allowed vlan 10,20
!
! --- Port utilisateur ---
interface GigabitEthernet0/1
 description Poste VPC8
 switchport mode access
 switchport access vlan 20
 spanning-tree portfast
 no shutdown
!
interface range GigabitEthernet1/0 - 3
 switchport mode access
 switchport access vlan 99
 shutdown
```

### 4.5 Configuration des postes de test

Sous VPCS :

```
VPC7> ip 192.168.10.11 255.255.255.0 192.168.10.254
VPC8> ip 192.168.20.11 255.255.255.0 192.168.20.254
```

Les postes pointent vers l'**adresse virtuelle HSRP**, jamais vers l'adresse réelle d'un routeur.
C'est tout l'intérêt du protocole : le poste ignore quel équipement le sert réellement et
continue de fonctionner quand celui-ci change.

---

## 5. Recette et tests

| # | Test | Commande ou méthode | Résultat attendu |
|---|---|---|---|
| 1 | Agrégation LACP | `show etherchannel summary` | Po1 en état (SU), deux ports en (P) |
| 2 | Négociation LACP | `show lacp neighbor` | Voisin détecté sur les deux liens |
| 3 | Trunks actifs | `show interfaces trunk` | VLAN 10 et 20 autorisés, natif 99 |
| 4 | Sous-interfaces routeur | `show ip interface brief` | Sous-interfaces .10 et .20 en up/up |
| 5 | Root bridge VLAN 10 | `show spanning-tree vlan 10` | Switch5 est root |
| 6 | Root bridge VLAN 20 | `show spanning-tree vlan 20` | Switch6 est root |
| 7 | État HSRP VLAN 10 | `show standby brief` | R4 Active, R3 Standby |
| 8 | État HSRP VLAN 20 | `show standby brief` | R3 Active, R4 Standby |
| 9 | Connectivité passerelle | `ping 192.168.10.254` depuis VPC7 | Succès |
| 10 | Routage inter-VLAN | `ping 192.168.20.11` depuis VPC7 | Succès |
| 11 | **Bascule HSRP** | Ping continu puis extinction de R4 | Reprise automatique par R3 |
| 12 | Retour à l'état nominal | Redémarrage de R4 | Reprise du rôle actif par préemption |
| 13 | Résilience de l'agrégation | Coupure d'un seul lien du Port-channel | Aucune perte, Po1 reste actif |

### Le test de bascule

C'est le test central du laboratoire, celui qui démontre que la redondance est réelle et non
théorique.

1. Lancer un ping continu depuis VPC7 vers 192.168.20.11 — le trafic transite par R4
2. Éteindre R4 pendant que le ping tourne
3. Observer les demandes expirées, puis la reprise du trafic
4. Relever le **nombre exact de paquets perdus**
5. Vérifier sur R3 que `show standby brief` affiche désormais l'état *Active*

Sous VPCS, la commande de ping continu est :

```
VPC7> ping 192.168.20.11 -t
```

Le nombre de paquets perdus constitue le résultat mesuré de ce laboratoire.

---

## 6. Difficultés rencontrées et enseignements

*Section à renseigner après réalisation, sur le modèle du premier case study : symptôme observé,
diagnostic, résolution, enseignement retenu.*

---

## 7. Améliorations envisagées

- Suivi d'interface HSRP (`standby track`) pour déclencher la bascule si la liaison montante
  tombe, et non uniquement si le routeur s'éteint
- Routage dynamique OSPF entre R4 et R3 via la liaison directe, en préparation d'une
  interconnexion vers un cœur de réseau
- Passage à VRRP pour un environnement multi-constructeurs
- Supervision SNMP des changements d'état HSRP, avec alerte à chaque bascule

---

[← Retour au portfolio](index.md)
