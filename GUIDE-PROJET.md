# Guide projet — Éditeur de cartes Extinction

## Description
Éditeur visuel de cartes pour le jeu **Aaaah!** de la plateforme **Extinction-MiniJeux** (extinction-minijeux.fr). Les cartes sont décrites dans un format XML custom. L'éditeur (`editor.html`) est une application web autonome (HTML/CSS/JS, sans framework).

## Liens utiles
- Site : https://extinction-minijeux.fr
- GitHub : https://github.com/extinction-minijeux/
- Aide : https://help.extinction-minijeux.fr
- Discord : https://discord.gg/2Ebe5C7czE
- Client open-source : https://github.com/extinction-minijeux/client-open-source

## Fichiers du projet
- `editor.html` — L'éditeur principal (fichier unique, ~54Ko)
- `code map` — Exemple de code XML d'une carte
- `tutoriel-extinction-editeur` — Tutoriel de Garganta sur le format XML et l'éditeur
- `screenshot.png` — Capture d'écran de référence
- `GUIDE-PROJET.md` — Ce fichier (guide de suivi)

---

## Spécifications techniques du jeu Aaaah!

### Constantes physiques
| Constante | Valeur | Description |
|---|---|---|
| IPS | 24 | Images par seconde |
| GRAVITE_X | 0.2 | Décélération horizontale |
| GRAVITE_Y | 0.4 | Gravité verticale (chute) |
| DEPLACEMENT_X | 2 | Vitesse horizontale (px/tick) |
| DEPLACEMENT_CONTA_X | 2.4 | Vitesse joueur contaminé |
| PUISSANCE_CRI_X | 4 | Poussée du cri (horizontal) |
| PUISSANCE_CRI_Y | -2 | Poussée du cri (vertical, vers le haut) |
| TEMPS_ENTRE_CRI | 10000ms | Cooldown entre les cris |
| TEMPS_ENTRE_SAUTS | 500ms | Cooldown entre les sauts |
| DUREE_PARTIE | 120000ms | Durée d'un round (2 minutes) |

### Points de collision du joueur (relatifs à sa position)
- **PD** (droite) : x+8, y+9
- **PG** (gauche) : x-10, y+9
- **PB** (bas) : x-1, y+19

### Zones de jeu
- Zone de jeu : ~800x400 pixels
- Origine : coin haut-gauche, X vers la droite, Y vers le bas
- **Zone de victoire** (infirmerie) : x > 740 ET y < 60
- **Zone de mort** : y > 380 OU (x > 740 ET y > 70)
- Offset map : DECALAGE_MAP = 1 pixel (x+1, y-1)

### Modes de jeu
| Mode | ID | Description |
|---|---|---|
| MODE_RUN | 0 | Standard |
| MODE_DEFILANTES | 1 | Maps défilantes |
| MODE_RALLY | 2 | Rally |
| MODE_FS | 3 | FS |
| MODE_MS | 4 | MS |

### Modes de mouvement (animations)
| Mode | ID | Description |
|---|---|---|
| Boucler | 0/1 | Boucle infinie |
| AllerRetour | 2 | Aller-retour |
| Stop | autre | Une seule fois |

### Système de collision
- Basé sur les pixels : `readPixels()` WebGL lit le frame buffer
- Tout pixel avec alpha ≠ 0 est une surface de collision
- Détection de couleur disponible : `detecterCouleur(x, y, r, g, b)` pour interactions spéciales

---

## Format XML des cartes

### Structure racine
```xml
<C N="NomDeLaCarte" A="NomAuteur" SAUT="1" VITESSE="1" GRAVITE="0.4">
  <!-- Premier enfant : groupes (formes jouables/collision) -->
  <G>
    <G P="x,y">          <!-- Sous-groupe positionné -->
      <L P="..." />       <!-- Formes à l'intérieur -->
      <R P="..." />
    </G>
  </G>
  <!-- Second enfant : fond (décorations, pas de collision) -->
  <F>
    <L P="..." />
    <P P="..." Z="..." />
    <CRI P="..." />       <!-- Cristaux -->
  </F>
</C>
```

### Balises de formes

#### `<L>` — Ligne
```
P="épaisseur,x1,y1,x2,y2"
```
- x1,y1 = point de départ
- x2,y2 = point d'arrivée **relatif au point de départ**

#### `<C>` — Courbe (Bézier quadratique)
```
P="épaisseur,x,y,cpX,cpY,toX,toY"
```
- x,y = point de départ
- cpX,cpY = point de contrôle (relatif)
- toX,toY = point d'arrivée (relatif)

#### `<R>` — Rectangle
```
P="épaisseur,x,y,largeur,hauteur,rempli"
```
- rempli : 1 = plein, 0 = contour uniquement

#### `<E>` — Ellipse
```
P="épaisseur,x,y,largeur,hauteur,rempli"
```
- rempli : 1 = plein, 0 = contour uniquement

#### `<P>` — Polygone
```
P="épaisseur,x,y,_,_,rempli" Z="x1,y1;x2,y2;x3,y3;..."
```
- Positions 4-5 du P inutilisées
- Z = sommets du polygone (paires séparées par des points-virgules)
- Le tracé commence à (0,0) puis suit les coordonnées Z

### Balises de mouvement (enfants des formes)

#### `<T>` — Translation
```
P="tempsDepart,tempsParcours,cibleX,cibleY,typeAnimation"
```
- tempsDepart/tempsParcours en secondes (×1000 en interne)
- typeAnimation : 1 = Boucle, 2 = Aller-retour, autre = Stop

#### `<R>` — Rotation (à l'intérieur d'une forme, ne pas confondre avec Rectangle)
```
P="tempsDepart,durée,degrés,typeAnimation"
```
- tempsDepart/durée en secondes (×1000 en interne)
- degrés = angle total de rotation
- typeAnimation : 1 = Boucle, 2 = Aller-retour, autre = Stop

### Groupes `<G>`
- Regroupent plusieurs formes ensemble
- Le mouvement appliqué au premier nœud XML du groupe se propage à tout le groupe
- Toutes les formes partagent un point pivot commun (centre de la bounding box)
- Translation/rotation du groupe déplace toutes les formes ensemble

### Balise spéciale `<CRI>` — Cristal
```
P="type,x,y,actif"
```
- type : type de cristal
- actif : true/false

### Mondes officiels
- 65 mondes (Monde_0 à Monde_64)
- Chaque monde a ses propres textures (spritesheets)
- Les mondes peuvent redéfinir les constantes physiques
- Les mondes peuvent avoir des interactions spéciales (ex: Monde_0 a un nuage qui souffle les joueurs avec vitesseX=-13, vitesseY=-8)

---

## Protocole réseau (WebSocket)

Format des messages : `Code#param1#param2#...`

### Messages Aaaah (serveur → client)
| Code | Description |
|---|---|
| IdL | Liste des joueurs au chargement de la map |
| IdO | Chargement nouvelle map/monde |
| IdIC | Init record et taux de survie |
| ChI | Mise à jour infos bloc |
| IdP | Données joueurs fin de map |
| IdJ | Nouveau joueur connecté |
| IdD | Joueur déconnecté |
| IdA | Projection (poussée par cri) |
| IdB | Cri |
| IdX | Mort d'un joueur |
| IdW | Victoire d'un joueur |
| MvG/MvD/MvH/MvS/MvA | Mouvement (gauche/droite/saut/stop/position absolue) |
| DeS/DeT | Dessin du guide (début/tracé) |
| ZoP/ZoC | Contamination (pré/contaminé) |
| EdV | Fin de map (vote) |

### Messages Aaaah (client → serveur)
| Code | Description |
|---|---|
| IdX | Signaler sa propre mort |
| IdW | Signaler sa propre victoire |
| DeS#x#y | Début trait du guide |
| DeT#x#y | Continuation trait du guide |

---

## Historique des modifications de l'éditeur

### 2026-03-23 — Sprites réels, sélection avancée, miroir, polygones, UI
- **Sprites CRI et TP** : remplacement des formes dessinées par les vrais PNG du jeu (cri.png 30x4, Teleporteur.png 35x6) en base64 inline
- **CRI "Inversé"** : renommage de "Visible" en "Inversé" dans les propriétés
- **Sélection rectangle** : clic+glisser sur le vide en mode sélection = rectangle de sélection bleu pointillé, toutes les formes touchées sont sélectionnées
- **Point central de sélection** : quand plusieurs formes sont sélectionnées, un point rouge central avec croix apparaît + cadre pointillé. Tirer ce point déplace toutes les formes
- **Multi-sélection** : Ctrl/Cmd+clic pour ajouter/retirer des formes de la sélection
- **Copier-coller** : Ctrl+C copie, Ctrl+V colle avec décalage de 15px
- **Ctrl+G** : raccourci clavier pour grouper la sélection
- **Dégrouper intelligent** : ne retire que la forme sélectionnée du groupe (pas le groupe entier). Groupe dissous si ≤1 forme restante
- **Miroir H/V** : crée une copie symétrique horizontale ou verticale (ne déplace pas l'original)
- **Shift+glisser** : contraint les lignes à horizontale, verticale ou 45°
- **Forme remplie par défaut** : checkbox dans la toolbar, appliquée aux rect/ellipse/polygone créés
- **Épaisseur par défaut** : dans la toolbar, persistante entre les créations
- **Polygones corrigés** : premier point cliqué = position de la forme (0,0 implicite du jeu), points suivants relatifs
- **Translation/Rotation masquées** pour CRI et TP (non supportés par le jeu)
- **Barre d'outils horizontale** : outils déplacés sous la map au lieu du panneau gauche
- **Panneau gauche réorganisé** : Groupes > Grille > Aimantation > Calque > Formes
- **Paramètres de carte dans le header** : Saut, Vitesse, Gravité inline dans la topbar
- **Thème adouci** : fond de carte gris moyen (#6b7080), panneaux gris-bleu, moins de fatigue visuelle
- **Spawn en noir** : meilleure lisibilité sur fond gris
- **Tooltips d'aide complétés** : toutes les sections mises à jour avec les nouvelles fonctionnalités

### 2026-03-21 — Téléporteurs, calque modèle, CRI rotatif, T0, handles
- **Téléporteurs `<TP>`** : nouvel outil (T), parsing/export XML `<TP TO="x,y" P="type,x,y" />`, rendu portail source (violet) + destination (vert) + flèche pointillée, handles pour déplacer source et destination, propriétés Dest. X/Y
- **Calque modèle** : import d'une image de référence (screenshot, croquis), opacité réglable, position/taille ajustables, bouton ajuster/supprimer, dessiné sous les formes, non exporté en XML
- **CRI rotatif** : forme diamant/cristal au lieu d'un cercle, propriété Angle (par pas de 15°), rendu avec reflet intérieur
- **Épaisseur min 1** : épaisseur minimum fixée à 1 (le jeu force T0→1), correction des || qui forçaient les valeurs falsy à 6
- **Déploiement** : fichier autonome, renommer en index.html et uploader sur un sous-domaine

### 2026-03-21 — Édition par handles + sprites + rendu fidèle
- **Édition par handles** : sélection améliorée, possibilité de tirer les points individuels des formes
  - Courbes : point de contrôle (courbure), point de départ, point d'arrivée
  - Lignes : chaque extrémité
  - Rectangles/Ellipses : 4 coins pour redimensionner
  - Polygones : chaque sommet individuellement
  - Corps de la forme : déplace tout
- **Rendu fidèle au jeu** : couleur noire, lineCap round (lignes/courbes), lineJoin miter (rectangles/ellipses), lineJoin round (polygones), épaisseur par défaut 3
- **Sprites PNG intégrés** : PasserelleDepart, BlocDroit, Infirmerie, TextBloc en base64 inline
- **Polygones** : ajout du point (0,0) initial comme le jeu, fill rule evenodd, champ fill lu depuis le XML
- **Rectangles négatifs** : gestion des dimensions négatives comme le jeu

### 2026-03-21 — Ajout des éléments fixes du jeu dans l'éditeur
- **Passerelle de départ** : rectangle à x=0, y=353, ~55x27px avec motif briques, label "PASSERELLE" et annotation "y=353 (descend)"
- **Bloc droit** : panneau sombre à x=693, y=44 avec labels info (Carte/Auteur, Guide, Timer, Joueurs)
- **Infirmerie** : zone verte à x=740, y=7-60 avec croix ✚ et label "INFIRMERIE" + coordonnées victoire
- **Zone de mort** : bande rouge semi-transparente en bas (y>380) avec label + zone mort droite (x>740, y>70)
- **Spawn joueur** : indicateur à x=60, y=320 avec silhouette stickman et cercle pointillé
- **Mécanique passerelle** : BriqueDepart.y = 353 + int(tempsÉcoulé/500), disparaît après 21.1 secondes

---
