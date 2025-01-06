# Système de Gestion de Bibliothèque Municipale

## 1. Diagramme de Contexte

```ascii
┌──────────────┐                              ┌──────────────────────┐
│    Abonné    │─────────────────────────────>│                      │
└──────────────┘     recherche/emprunte       │                      │
                                              │                      │
┌──────────────┐                              │  Système de Gestion  │
│Bibliothécaire│─────────────────────────────>│   de Bibliothèque   │
└──────────────┘     gère catalogue           │                      │
                                              │                      │
┌──────────────┐                              │                      │
│  Directeur   │─────────────────────────────>│                      │
└──────────────┘     administre               └──────────────────────┘
```

## 2. Diagramme des Cas d'Utilisation

### Acteurs identifiés :
1. **Abonné**
2. **Bibliothécaire**
3. **Directeur**

### Cas d'utilisation principaux :

```ascii
┌────────────────────────────────────────────────────┐
│           Système de Gestion Bibliothèque          │
│                                                    │
│  ┌─────────────────────┐                          │
│  │ Rechercher un livre │<──────── Abonné          │
│  └─────────────────────┘                          │
│                                                    │
│  ┌─────────────────────┐                          │
│  │ Emprunter un livre  │<──────── Abonné          │
│  └─────────────────────┘                          │
│                                                    │
│  ┌─────────────────────┐                          │
│  │  Réserver un livre  │<──────── Abonné          │
│  └─────────────────────┘                          │
│                                                    │
│  ┌─────────────────────┐                          │
│  │  Gérer les prêts    │<──────── Bibliothécaire  │
│  └─────────────────────┘                          │
│                                                    │
│  ┌─────────────────────┐                          │
│  │  Gérer catalogue    │<──────── Bibliothécaire  │
│  └─────────────────────┘                          │
│                                                    │
│  ┌─────────────────────┐                          │
│  │ Consulter stats     │<──────── Directeur       │
│  └─────────────────────┘                          │
└────────────────────────────────────────────────────┘
```

## 3. Diagramme de Classes

```ascii
┌───────────────┐      ┌───────────────┐      ┌───────────────┐
│     Livre     │1    n│   Exemplaire  │0..1  │     Prêt      │
├───────────────┤──────├───────────────┤──────├───────────────┤
│ codeCatalogue │      │ cote          │      │ datePret      │
│ titre         │      │ dateAcquis    │      │ dateRetour    │
│ theme         │      │ codeUsure     │      │ dateRetourPrev│
├───────────────┤      ├───────────────┤      ├───────────────┤
│ +rechercher() │      │ +majUsure()   │      │ +renouveler() │
└───────┬───────┘      └───────────────┘      └───────┬───────┘
        │                                             │
        │                                             │
    n   │                                         1   │
┌───────┴───────┐      ┌───────────────┐            │
│    MotClé     │      │    Abonné     │            │
├───────────────┤      ├───────────────┤            │
│ libelle       │      │ matricule     │◄───────────┘
└───────────────┘      │ nom           │
                       │ adresse       │
                       │ dateAdhesion  │
                       └───────────────┘
```

## 4. Contraintes OCL

```ocl
-- Contraintes sur les Prêts
context Pret
-- Durée maximale d'un prêt est de 15 jours
inv duree_pret_15_jours:
    self.dateRetourPrevue = self.datePret + 15

-- Date de retour effective doit être postérieure à la date de prêt
inv date_retour_valide:
    self.dateRetourEffective.isDefined() implies 
    self.dateRetourEffective >= self.datePret

-- Contraintes sur les Exemplaires
context Exemplaire
-- Unicité des cotes des exemplaires
inv cote_unique:
    Exemplaire.allInstances()->forAll(e1, e2 | 
        e1 <> e2 implies e1.cote <> e2.cote)

-- Code d'usure valide
inv code_usure_valide:
    Set{'Neuf', 'Bon', 'Usé', 'Mauvais'}->includes(self.codeUsure)

-- Contraintes sur les Réservations
context Reservation
-- Validité d'une réservation (7 jours après retour)
inv validite_reservation:
    self.dateLimiteValidite = self.dateReservation + 7

-- Un livre ne peut être réservé que s'il est déjà emprunté
inv reservation_livre_emprunte:
    self.livre.exemplaires->exists(e | e.pret->notEmpty())

-- Contraintes sur les Abonnés
context Abonne
-- Limite du nombre d'emprunts simultanés
inv limite_emprunts:
    self.prets->select(p | p.dateRetourEffective.isUndefined())->size() <= 5

-- Age minimum pour l'inscription
inv age_minimum:
    let age: Integer = today().year - self.dateNaissance.year in
    age >= 7

-- Contraintes sur les Livres
context Livre
-- Un livre doit avoir au moins un mot-clé
inv minimum_mot_cle:
    self.motsCles->size() >= 1

-- Un livre doit avoir au moins un auteur
inv minimum_auteur:
    self.auteurs->size() >= 1

-- Le code catalogue doit être unique
inv code_catalogue_unique:
    Livre.allInstances()->forAll(l1, l2 | 
        l1 <> l2 implies l1.codeCatalogue <> l2.codeCatalogue)
```

## 5. Diagramme de Composants

```ascii
┌─────────────────────────────────────────┐
│            Interface Utilisateur         │
└───────────────────┬─────────────────────┘
                    │
                    ▼
┌─────────────────────────────────────────┐
│ ┌─────────────┐  ┌─────────────┐  ┌───┐ │
│ │  Gestion    │  │  Gestion    │  │   │ │
│ │  des Prêts  │  │  Catalogue  │  │BD │ │
│ └─────────────┘  └─────────────┘  └───┘ │
└─────────────────────────────────────────┘
```

## 6. Diagramme de Déploiement

```ascii
┌──────────────┐     HTTP     ┌──────────────────┐
│ Poste Client │──────────────►│Serveur Application│
└──────────────┘              └────────┬─────────┘
                                      │
                                      │ SQL
                                      ▼
                             ┌──────────────────┐
                             │   Serveur BD     │
                             └──────────────────┘
```

## Spécifications Techniques

1. **Volumétrie**
   - 36 872 livres
   - 21 709 titres différents
   - 2 634 abonnés

2. **Règles de Gestion**
   - Durée de prêt : 15 jours
   - Réservation valide 7 jours après retour
   - Identification unique des exemplaires par cote
   - Suivi de l'état d'usure des livres

3. **Fonctionnalités de Recherche**
   - Par titre
   - Par auteur
   - Par éditeur
   - Par thème
   - Par mots-clés