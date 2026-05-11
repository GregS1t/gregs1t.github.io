---
layout: course
title: Algorithmique et Programmation L2
description: Fondamentaux de l'algorithmique et de la programmation en Python, depuis les bases du langage jusqu'aux méthodes numériques appliquées à la physique (intégration, équations différentielles, ajustement de données, nombres aléatoires).
instructor: Grégory Sainton (Assistant TP)
year: 2020

location: Université de Paris, UFR de Physique
course_id: IF402-algorithmique-programmation

schedule:
  - week: 1
    topic: Introduction à l'algorithmique et à Python
    description: Organisation de l'UE, notion d'algorithme et pseudo-langage, premiers pas avec JupyterLab, variables, opérateurs et bibliothèques.

  - week: 2
    topic: Fonctions, boucles et branchements
    description: Préambule algorithmique, retour sur les variables, définition et appel de fonctions, structures conditionnelles (if/elif/else), boucles (for, while), introduction à NumPy.

  - week: 3
    topic: Fichiers, représentation des nombres et visualisation
    description: Lecture et écriture de fichiers (NumPy, texte, CSV), représentation binaire et virgule flottante, représentation graphique avec Matplotlib, introduction au calcul formel avec SymPy.

  - week: 4
    topic: Images, tableaux et complexité algorithmique
    description: Représentation numérique des images, manipulation de tableaux NumPy, notion de complexité algorithmique (O(N), O(N²), O(N log N)), algorithmes de tri (sélection, insertion, quicksort).

  - week: 5
    topic: Recherche et résolution de f(x)=0
    description: Recherche séquentielle, dichotomique et par interpolation, méthodes de résolution numérique (balayage, dichotomie, Newton-Raphson), retour sur les images.

  - week: 6
    topic: Intégration numérique
    description: Méthodes de quadrature (rectangles, point milieu, trapèzes, Simpson, Romberg), estimation et comparaison des erreurs de convergence.

  - week: 7
    topic: Équations différentielles ordinaires
    description: Méthodes aux différences finies, schémas d'Euler explicite et implicite, méthodes d'ordre supérieur (point milieu, Heun, Runge-Kutta d'ordre 4).

  - week: 8
    topic: Nombres aléatoires
    description: "Distanciel. Générateurs pseudo-aléatoires, congruences linéaires, Mersenne Twister, lois de probabilité (uniforme, exponentielle), méthode d'inversion de la fonction de partition."

  - week: 9
    topic: Ajustement de données
    description: "Distanciel. Méthode des moindres carrés, régression linéaire, minimisation du chi-2, introduction à scipy.optimize pour les problèmes non linéaires."

---

## Objectifs du cours

Ce cours d'introduction à l'informatique scientifique vise à donner aux étudiants en physique les outils nécessaires pour résoudre des problèmes physiques par le calcul numérique. À l'issue de ce cours, les étudiants seront capables de :

- Concevoir et implémenter des algorithmes pour résoudre des problèmes physiques
- Programmer en Python avec les bibliothèques NumPy et Matplotlib
- Appliquer les méthodes numériques classiques (intégration, résolution d'équations, équations différentielles)
- Analyser la complexité algorithmique d'un programme
- Traiter et visualiser des données expérimentales

## Prérequis

- Notions de mathématiques de lycée (fonctions, dérivées, probabilités)
- Aucune expérience préalable en programmation n'est requise

## Évaluation (6 ECTS)

- Questions de cours hebdomadaires (EC) : 10 % de la note finale
- Travaux pratiques (ETP) : 20 % de la note finale
- Évaluation partielle mi-semestre (EP) : 35 % de la note finale
- Examen final (EF) : 35 % de la note finale
- Note finale : 70 % × MAX((EP+EF)/2, EF) + 20 % × ETP + 10 % × EC

## Références

- Documentation officielle NumPy : https://numpy.org/doc/
- Documentation officielle Matplotlib : https://matplotlib.org/
