# Design : Assistant Tarot - Compteur de points

## Objectif

Web app mobile-first pour compter les points au tarot (jeu de cartes français), conforme aux règles officielles de la FFT. Deux modes : calculette rapide et suivi de partie complète.

## Stack technique

Un seul fichier `index.html` (HTML + CSS + JS intégrés). Pas de framework, pas de build. Persistance via `localStorage`.

## Architecture

Deux onglets principaux :
- **Nouvelle donne** : formulaire de saisie pour calculer le score d'une donne
- **Partie en cours** : tableau des scores cumulés, historique, classement

## Configuration de partie

- Nombre de joueurs : 3, 4 ou 5
- Noms des joueurs (pré-remplis "Joueur 1", "Joueur 2"...)

## Formulaire "Nouvelle donne"

Champs de saisie :
1. Preneur (sélection parmi les joueurs)
2. Partenaire appelé (uniquement à 5 joueurs)
3. Contrat : Prise / Garde / Garde Sans / Garde Contre
4. Nombre de bouts du preneur : 0 / 1 / 2 / 3
5. Points du preneur : 0 à 91
6. Petit au bout : Non / Attaque / Défense
7. Poignée : Non / Simple / Double / Triple (attaque ou défense)
8. Chelem : Non / Annoncé réussi / Non annoncé réussi / Annoncé chuté

Nombre d'atouts pour la poignée selon le nombre de joueurs :
- 3 joueurs : Simple 13 / Double 15 / Triple 18
- 4 joueurs : Simple 10 / Double 13 / Triple 15
- 5 joueurs : Simple 8 / Double 10 / Triple 13

Résumé du calcul affiché en temps réel avant validation.

## Logique de calcul

### Score de base

```
score_base = (25 + |points_preneur - seuil|) * multiplicateur
```

Seuils selon bouts : 0 → 56, 1 → 51, 2 → 41, 3 → 36

Multiplicateurs : Prise ×1, Garde ×2, Garde Sans ×4, Garde Contre ×6

Si points_preneur < seuil → contrat chuté, score négatif pour le preneur.

### Primes

- Petit au bout : 10 × multiplicateur (pour le camp qui l'a réalisé)
- Poignée : Simple 20 / Double 30 / Triple 40 (valeur fixe, pour le camp vainqueur)
- Chelem : annoncé réussi +400 / non annoncé réussi +200 / annoncé chuté -200

### Répartition entre joueurs

- 3 joueurs : preneur ×2, chaque défenseur ×(-1)
- 4 joueurs : preneur ×3, chaque défenseur ×(-1)
- 5 joueurs : preneur ×2, partenaire appelé ×1, chaque défenseur ×(-1)

La somme des scores est toujours égale à 0.

## Écran "Partie en cours"

- Tableau des scores cumulés (colonnes = joueurs, lignes = donnes, total en bas)
- Classement des joueurs par score décroissant
- Détail d'une donne au tap sur une ligne
- Bouton "Annuler la dernière donne"
- Bouton "Nouvelle partie" (avec confirmation)

## Design

- Mobile-first, gros boutons tactiles
- Palette sobre inspirée des couleurs du jeu de cartes
- Responsive (utilisable aussi sur desktop)
