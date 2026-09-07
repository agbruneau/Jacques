# Scénarios d'implantation : une application locale CoramDeo en Go ou en Rust, sur Windows 11

Analyse d'architecture. Version 1, 7 septembre 2026. Statut : proposition, décision à prendre par l'auteur.

Marquage de certitude employé dans tout le document :

- **[confirmé]** : vérifié en ligne le 7 septembre 2026 à la source citée en annexe A, ou constaté dans le dépôt.
- **[probable]** : savoir de formation cohérent avec les sources consultées, non revérifié.
- **[hypothèse]** : choix de travail ou estimation propre à cette analyse.
- **[à vérifier]** : à confirmer sur le poste cible avant de trancher.

---

## 1. Conclusion

Recommandation : bâtir en **Go** un exécutable unique, `coramdeo.exe`, qui réunit une ligne de commande, un serveur local et un serveur MCP. Il remplace l'outillage Python de production, produit les trois livrables (Markdown, HTML, PDF) à partir d'une **source HTML unique** en imprimant le PDF par le moteur Chromium d'Edge déjà présent sur Windows 11, indexe le corpus dans SQLite FTS5 et sert le site localement. L'extraction du PDF *NEG - MacArthur* reste en Python : elle a déjà tourné et ne retourne pratiquement jamais.

Compromis principal : on renonce à un moteur de composition natif, donc à la maîtrise fine de la typographie (césure, veuves et orphelines, en-têtes courants), et on dépend d'Edge au moment de produire un PDF. En échange, la chaîne d'outils sur Windows se réduit à un installateur Go sans compilateur C, l'itération est rapide, et la **parité HTML/PDF/Markdown est obtenue par construction**, alors que les skills l'exigent aujourd'hui à la main.

Alternative sérieuse : **Rust avec Typst embarqué** comme moteur de composition (option B, §7). Elle produit de meilleurs PDF, sans processus externe, avec une sortie déterministe, et ouvre la voie à une application de bureau Tauri. Elle coûte une chaîne d'outils MSVC de plusieurs gigaoctets, des compilations longues et un troisième gabarit à maintenir à côté du HTML et du Markdown.

Conditions qui renversent la recommandation (§8.3) : une exigence typographique que Chromium ne satisfait pas ; l'impossibilité d'automatiser Edge sur le poste (régression connue depuis Edge 141, à tester en premier) ; la volonté d'une application de bureau distribuable à d'autres postes ; ou la décision de consolider tout l'outillage local sur Rust, vraisemblablement déjà présent sur la machine par RuVector [probable].

---

## 2. Hypothèse de travail et périmètre

La demande ne nomme pas la fonction de l'application. Le dépôt la désigne assez nettement : le contenu théologique est produit par deux skills Claude Code (`sermon-research`, `sermon-series`) qui délèguent la mise en forme à des scripts Python, le site est un ensemble de pages HTML autonomes sans étape de construction, et aucun outil ne permet de vérifier ni d'interroger un corpus qui dépasse aujourd'hui trois millions et demi de mots. L'application analysée ici est donc **l'outil local de production et de consultation du dépôt CoramDeo**. [hypothèse]

Périmètre fonctionnel retenu, avec la trace correspondante dans le dépôt :

| # | Capacité | Ce qui existe aujourd'hui | Où dans le dépôt |
|---|---|---|---|
| F1 | Rendu des livrables : JSON structuré vers PDF et Markdown, plus présentation HTML | `generate-pdf.py` (reportlab) dans chaque skill ; HTML adapté à la main depuis `template-presentation.html` | `.claude/skills/sermon-research/`, `.claude/skills/sermon-series/`, `.claude/shared/pdf_utils.py` |
| F2 | Rendu d'un Markdown doctrinal en PDF maison | `md-to-pdf.py` (reportlab), parseur volontairement partiel | `3 - Pérégrination/00 - Avant-propos/md-to-pdf.py` |
| F3 | Extraction du texte NEG et des notes JMA depuis le PDF source | `extract_nt.py`, `extract_at.py` (PyMuPDF), classification des spans par police et taille | même dossier |
| F4 | Vérification du corpus : liens, structure des dossiers, encodage, attributions | Aucun outil ; fait par des agents (voir `gauntlet-log.md`) | racine |
| F5 | Consultation et recherche plein texte | Aucun outil local ; le site est statique | 925 `index.html`, 1 016 `.md` |
| F6 | Point d'intégration avec Claude Code | Les skills lancent `python assets/generate-pdf.py <json>` | `SKILL.md` des deux skills |

Hors périmètre : la production du contenu théologique (elle reste aux skills et au modèle), l'hébergement du site (statique, inchangé) et toute synchronisation entre plusieurs postes.

---

## 3. Contraintes constatées dans le dépôt [confirmé]

| Fait | Mesure | Conséquence pour l'application |
|---|---|---|
| Volume | 2 980 fichiers ; 1 016 `.md` (887 recherches de péricope, 31 séries, 30 recherches globales, 30 NEG, 30 JMA) ; 954 PDF totalisant 113 Mo ; 925 `index.html` | L'indexation se compte en secondes. Les PDF pèsent la moitié de l'arbre de travail |
| Corpus textuel | Environ 3,7 millions de mots dans les `.md` | SQLite FTS5 suffit largement ; aucun moteur de recherche externe n'est justifié |
| Chemins | Espaces, accents, parenthèses et points dans les noms (`... (Jude 1.1-2)/`) ; chemin relatif le plus long : 170 caractères | Sous `C:\Users\<nom>\...`, on reste sous la limite historique de 260 caractères, avec peu de marge. L'outil doit tolérer les chemins longs et les noms Unicode |
| Encodage des liens | Les `href` de `index.html` sont percent-encodés (`3%20-%20P%C3%A9r%C3%A9grination/...`) | Le vérificateur de liens doit décoder, normaliser en NFC, puis résoudre sur le système de fichiers |
| PDF sous git | `.gitattributes` force `*.pdf binary` après une corruption par conversion CRLF | L'auteur travaille sous Windows avec conversion des fins de ligne. Tout nouveau générateur doit produire des PDF **stables** (pas de régénération sans changement de contenu) pour ne pas gonfler l'historique |
| Traces Windows | `.gitignore` exclut `* - Copie*` | Machine cible confirmée par les artefacts |
| Outillage local existant | `.gitignore` exclut `ruvector.db`, décrit comme base de l'outillage local Ruflo / RuVector | RuVector est écrit en Rust [probable] ; une chaîne Rust ou Node est vraisemblablement déjà installée sur le poste |
| Dépendances Python | reportlab, PyMuPDF ; polices TTF Crimson Pro et Arsenal SC (licence OFL) dans `.claude/shared/fonts/` | Ces polices peuvent être embarquées dans le binaire. `pdf_utils.py` cherche en plus Times New Roman sur la machine pour les macrons de translittération (ō, ē), ce qui rend le rendu dépendant du poste |
| Contrat des skills | JSON, puis `generate-pdf.py` produit PDF et Markdown ; HTML adapté du gabarit à la main ; « parité de contenu » exigée entre les trois sorties | Le point de substitution tient en une ligne de commande dans chaque `SKILL.md` |
| Deux styles PDF | `pdf_utils.py` (« Quiet Doctrine », marine et or) et `md-to-pdf.py` (brun et orange) | À unifier, ou à conserver comme deux feuilles de style d'un même moteur |

---

## 4. Scénarios de forme, indépendants du langage

| Scénario | Ce que l'auteur obtient | Dépendances sur Windows 11 | Coût propre | Verdict |
|---|---|---|---|---|
| **S1. Ligne de commande** : un `.exe` avec sous-commandes `render`, `check`, `index`, `search`, `serve`, `mcp` | Remplacement direct des scripts Python ; appel depuis les skills ; scriptable dans PowerShell et le Planificateur de tâches | Aucune | Le plus bas | **Retenu** comme socle |
| **S2. Serveur local + navigateur** : `coramdeo serve --open` sert le site tel quel et ajoute recherche et tableaux de bord | Une interface de consultation sans aucune boîte à outils graphique ; Edge lancé avec `--app=http://localhost:<port>` donne une fenêtre sans barre d'adresse [probable] | Edge (composant du système) | Bas : HTML et CSS déjà définis par le système visuel du site | **Retenu** comme surface de lecture |
| **S3. Application de bureau WebView2** : Tauri (Rust) ou Wails (Go) | Fenêtre native, barre d'état, boîtes de dialogue, installateur MSI ou NSIS | WebView2 Evergreen, préinstallé sur Windows 11 [confirmé] | Élevé : empaquetage, signature, mises à jour, deuxième dépôt de code frontal | **Différé** : rien dans le périmètre ne l'exige |
| **S4. Serveur MCP local** : les mêmes capacités exposées comme outils à Claude Code, en transport stdio | Les skills appellent `render`, `check`, `search` sans passer par un interpréteur ni par le shell ; sorties structurées | Aucune | Bas : SDK officiels en Go et en Rust [confirmé] | **Retenu**, dans le même binaire que S1 |

Sur S3 : Tauri en est à la version 2.11.5 (1er juillet 2026) et s'appuie sur WebView2 sous Windows [confirmé]. Wails v3 est en bêta (bêta 9, août 2026) et Wails v2 reste la version stable [confirmé]. Si une coque de bureau devient nécessaire, l'écart de maturité favorise Rust et Tauri ; c'est l'une des conditions de renversement du §8.3.

Combinaison retenue : **S1 + S4 dans un seul binaire, avec S2 comme surface de consultation**. S3 n'est pas exclu, il est reporté jusqu'à ce qu'un besoin le justifie.

---

## 5. Le nœud technique : produire le PDF

Le PDF est la capacité qui départage réellement les options. Les autres briques (Markdown, gabarits HTML, SQLite, HTTP, MCP) existent à qualité comparable dans les deux langages.

### 5.1 Trois voies

**(a) Bibliothèque PDF native du langage.**

- Go : `go-pdf/fpdf` a quitté GitHub (dépôt archivé) pour Codeberg, où il est actif (dernier commit du 27 mai 2026, polices TTF UTF-8, justification « J ») [confirmé]. `signintech/gopdf` (MIT) offre polices TTF, justification et tableaux [confirmé]. `maroto` v2 est stable mais dépend encore du `gofpdf` archivé [confirmé]. Limite commune [probable] : aucune de ces bibliothèques ne compose un paragraphe à styles mêlés (gras et italique dans la même ligne, comme le mini-HTML de `Paragraph` chez reportlab), aucune ne coupe les mots, et la pagination des tableaux se fait à la main. Porter fidèlement `pdf_utils.py` revient à écrire un petit moteur de mise en page.
- Rust : `printpdf` est de bas niveau ; `genpdf`, qui le surmonte, est d'une maintenance incertaine [à vérifier]. Même diagnostic.

**(b) Typst comme moteur de composition.** Typst 0.15.1 (17 juillet 2026) s'embarque comme bibliothèque Rust sous licence Apache-2.0 ; la caisse `typst-kit` a été refondue pour faciliter l'implémentation du `World` requis [confirmé]. Depuis Go, on l'appelle comme exécutable externe (`typst.exe`, disponible par winget) [probable]. Typst apporte justification, césure française, tableaux à en-tête répété, en-têtes et pieds courants, polices embarquées et sortie déterministe [probable]. Le gabarit s'écrit en langage Typst et lit le JSON du livrable directement.

**(c) HTML vers PDF par le moteur Chromium d'Edge.** Un seul document HTML, une feuille de style écran (le thème sombre du site) et une feuille de style impression (le style clair des PDF actuels). Chromium sous Windows coupe les mots en français lorsque `lang="fr"` est posé et `hyphens: auto` activé [confirmé]. Les en-têtes et pieds de page avec numérotation passent par les gabarits de `Page.printToPDF` du protocole DevTools [probable].

Risque propre à (c), à traiter le premier jour : depuis Edge 141.0.3537.57 (octobre 2025), le drapeau `--print-to-pdf` en mode headless ne fonctionne plus tel quel ; les contournements rapportés sont `--headless=old`, l'ajout de `--user-data-dir` et l'usage de barres obliques dans les chemins [confirmé]. Le comportement de la méthode `Page.printToPDF` du protocole DevTools, qu'emploient les bibliothèques d'automatisation (chromedp en Go, chromiumoxide ou headless_chrome en Rust), n'est pas documenté dans ces fils [à vérifier]. Conséquence de conception : piloter Edge par le protocole DevTools plutôt que par le drapeau de ligne de commande, et valider cette voie sur le poste avant tout autre travail.

### 5.2 Comparaison des trois voies

| Critère | (a) Bibliothèque native | (b) Typst | (c) Chromium / Edge |
|---|---|---|---|
| Typographie (justification, césure, tableaux, en-têtes courants) | Faible : à programmer | Excellente | Bonne : césure française, `orphans` / `widows`, `break-inside: avoid` sur les tableaux [probable] |
| Parité HTML / PDF / Markdown | Trois rendus distincts | Deux gabarits (Typst, HTML) | **Une seule source** ; le Markdown se dérive du même modèle |
| Dépendance à l'exécution | Aucune | Aucune (Rust) ou `typst.exe` (Go) | `msedge.exe`, composant de Windows 11 |
| Déterminisme du PDF | Contrôlable | Natif (`set document(date: none)`) [probable] | Chromium écrit une date de création et un identifiant ; à neutraliser par régénération conditionnée au hachage des entrées [hypothèse] |
| Vitesse | Instantanée | Sous la seconde | 0,2 à 1 s par document avec une instance de navigateur réutilisée [hypothèse] |
| Effort pour égaler `pdf_utils.py` | Élevé | Moyen (nouveau langage de gabarit) | Faible (CSS d'impression sur un gabarit qui existe déjà) |
| Risque d'écosystème | `gofpdf` archivé, fork déplacé | Caisse en 0.x, API mobile entre versions [probable] | Régression Edge 141 sur le mode headless |

### 5.3 L'extraction du PDF source (F3) : ne pas porter

PyMuPDF donne, par span, la police, la taille et la position ; c'est exactement ce qu'exploitent `extract_nt.py` et `extract_at.py`. Les équivalents Go purs sont faibles sur les encodages et les CMaps [probable] ; en Rust, `pdfium-render` exige la DLL PDFium et `mupdf-rs` compile MuPDF (AGPL) [probable]. Les 60 fichiers `NEG - *.md` et `JMA - *.md` sont déjà dans le dépôt et la source, *La Bible d'étude MacArthur*, ne change pas. Décision : l'extraction reste en Python, appelée au besoin ; si l'on veut vraiment retirer Python du poste un jour, `mutool draw -F stext` produit un XML par span avec police et taille et se pilote depuis n'importe quel langage [probable].

---

## 6. Go contre Rust sur les préoccupations concrètes de ce projet

| Préoccupation | Go | Rust | Lecture |
|---|---|---|---|
| Version courante | 1.27, août 2026 [confirmé] | 1.98, août 2026 [confirmé] | Les deux sont matures et à cadence régulière |
| Installation sur Windows 11 | Installateur MSI ou winget ; aucun compilateur C si CGO est désactivé [probable] | `rustup`, plus « MSVC v143 build tools » et « Windows 11 SDK » pour la cible `x86_64-pc-windows-msvc` [confirmé] ; cible `-gnu` ou `-gnullvm` sinon [probable] | Avantage Go : un seul installateur |
| Compilation croisée depuis Linux ou macOS (Claude Code sur le web compile côté serveur) | Triviale : `GOOS=windows GOARCH=amd64` [confirmé] | Possible, avec `cargo-xwin` ou la cible gnu [probable] | Avantage Go |
| Temps de compilation | Secondes | Minutes pour un binaire qui embarque Typst [probable] | Avantage Go, sensible en développement assisté par agent |
| Taille du binaire | 10 à 20 Mo | 5 à 40 Mo selon Typst et les polices embarquées | Indifférent [hypothèse] |
| PDF natif (voie a) | `fpdf` sur Codeberg, `gopdf` : justification oui, styles mêlés non | `printpdf`, `genpdf` : idem | Égalité basse |
| Typst (voie b) | Processus externe | Bibliothèque embarquée [confirmé] | Avantage net Rust |
| Chromium par DevTools (voie c) | `chromedp` | `chromiumoxide`, `headless_chrome` | Égalité [probable] |
| Markdown | `goldmark` (CommonMark, tableaux, notes) | `pulldown-cmark`, `comrak` | Égalité [probable] |
| Gabarits HTML | `html/template`, bibliothèque standard, échappement contextuel | `minijinja`, `askama`, `tera` | Égalité ; léger avantage Go (rien à ajouter) |
| SQLite et FTS5 | `modernc.org/sqlite` v1.58.0 (1er septembre 2026), SQLite 3.53.4, pur Go, `windows/amd64` et `windows/arm64` [confirmé] ; FTS5 inclus [probable] | `rusqlite` avec `bundled` compile SQLite en C, donc exige la chaîne MSVC [probable] | Avantage Go : aucune compilation C |
| Index de recherche dédié | `bleve` | `tantivy` | Inutile à 3,7 M de mots ; SQLite FTS5 suffit [hypothèse] |
| Serveur HTTP local | `net/http`, bibliothèque standard | `axum` + `tower-http` | Léger avantage Go |
| SDK MCP | SDK officiel `modelcontextprotocol/go-sdk` v1.6, maintenu avec Google [confirmé] | `rmcp`, SDK officiel, spécification 2026-07-28 [confirmé] | Égalité |
| Coque WebView2 | Wails v2 stable, v3 en bêta [confirmé] | Tauri 2.11.5 [confirmé] | Avantage Rust |
| Chemins longs Windows | `os` ajoute `\\?\` automatiquement et s'en dispense quand le système accepte les chemins longs (`os/path_windows.go`) [confirmé] | La bibliothèque standard fait de même [probable] | Égalité |
| Console UTF-8 (accents dans PowerShell) | Basculer la page de code en 65001 via `x/sys/windows` [probable] | Idem via la caisse `windows` [probable] | Égalité ; Windows Terminal règle la plupart des cas |
| Faux positifs Microsoft Defender | Binaires Go signalés par le modèle d'apprentissage de Defender (Wacatac, Sabsik), problème récurrent et ouvert [confirmé] | Occasionnel [probable] | Léger désavantage Go ; un binaire compilé localement est rarement touché, et une exclusion de dossier suffit sur un poste personnel [probable] |
| Développement assisté par Claude Code | Boucle compiler-tester très courte ; langage petit | Le compilateur attrape davantage avant l'exécution ; boucle plus longue | Avantage Go pour l'itération, Rust pour la sûreté [hypothèse] |

---

## 7. Trois architectures candidates et leur évaluation

**A. Go, source HTML unique, PDF par Edge (DevTools), SQLite FTS5, MCP.** Modèle de données commun (le JSON des skills), un gabarit HTML par type de livrable avec feuille écran et feuille impression, dérivation du Markdown depuis le même modèle, `chromedp` pour le PDF, `modernc.org/sqlite` pour la recherche, `net/http` pour `serve`, SDK MCP officiel pour `mcp`.

**B. Rust, Typst embarqué, axum, rusqlite, rmcp, Tauri en option.** Gabarits Typst pour le PDF, gabarits HTML pour le site, Markdown dérivé du modèle. Aucune dépendance à l'exécution. Coque Tauri possible ensuite sans changer de langage.

**C. Go, Typst en exécutable externe.** L'architecture A avec `typst.exe` à la place d'Edge pour le PDF. Meilleure typographie, mais deux gabarits (Typst et HTML) et une dépendance installée par winget.

Grille d'évaluation. Les poids sont ceux de cette analyse et constituent le levier de décision ; notes de 1 (faible) à 5 (fort). [hypothèse]

| Critère | Poids | A | B | C |
|---|---|---|---|---|
| Qualité typographique du PDF | 3 | 3 | 5 | 5 |
| Parité HTML / PDF / Markdown par construction | 3 | 5 | 2 | 2 |
| Simplicité de la chaîne d'outils sur Windows | 2 | 5 | 2 | 4 |
| Absence de dépendance à l'exécution | 2 | 3 | 5 | 3 |
| Effort initial | 2 | 4 | 2 | 4 |
| Coût de maintenance (gabarits, écosystème) | 3 | 4 | 3 | 3 |
| Chemin vers une application de bureau | 1 | 2 | 5 | 2 |
| Risque d'écosystème ou de régression | 2 | 3 | 4 | 4 |
| **Total pondéré (maximum 90)** | | **68** | **61** | **62** |

Sensibilité : le classement bascule vers B ou C dès que la typographie pèse nettement plus que la parité (par exemple 5 contre 1). Autrement dit, la question à trancher par l'auteur n'est pas « Go ou Rust » mais : **la finesse typographique du PDF vaut-elle davantage que l'unicité de la source ?** Cette analyse répond non, pour un livrable lu à l'écran ou imprimé au bureau ; elle répondrait oui pour une publication.

---

## 8. Recommandation, compromis, conditions de renversement

### 8.1 Recommandation

L'architecture A. Go pour le langage ; un binaire `coramdeo.exe` ; six sous-commandes (`render`, `check`, `index`, `search`, `serve`, `mcp`) ; la source HTML unique comme principe structurant ; Edge piloté par le protocole DevTools pour le PDF ; SQLite FTS5 pour la recherche ; l'extraction laissée à Python.

### 8.2 Compromis assumés

1. **Typographie.** Chromium ne fait pas de composition de livre. Il coupe les mots en français, gère veuves et orphelines et évite de scinder un tableau, ce qui couvre les besoins d'une recherche exégétique de dix pages. Il ne fera ni notes de bas de page natives, ni en-têtes courants portant le titre de section, sans détours.
2. **Dépendance à Edge au moment du rendu.** Acceptable sur un poste Windows 11 où Edge est un composant du système. Inacceptable si l'outil doit tourner sur un serveur ou un poste verrouillé ; voir §8.3.
3. **Stabilité des PDF sous git.** Chromium date chaque PDF ; l'outil ne régénère un PDF que si le hachage de ses entrées (JSON, gabarit, feuille de style) a changé, ce qui supprime les diffs parasites sans post-traitement du fichier.
4. **Faux positifs Defender.** Risque réel mais faible pour un binaire compilé localement ; une exclusion du dossier de sortie règle le cas résiduel.

### 8.3 Conditions qui renversent la recommandation

| Si... | Alors... |
|---|---|
| Le test du premier jour montre que `Page.printToPDF` échoue sur Edge 141 et suivants et que les contournements ne tiennent pas | Passer à C (Go + `typst.exe`) sans toucher au reste de l'architecture ; A et C partagent tout sauf le module PDF |
| Les livrables doivent atteindre une qualité de publication (notes de bas de page, en-têtes courants, contrôle fin de la césure) | Passer à B ou à C ; Typst est le seul moteur des trois voies qui le fait sans programmation de mise en page |
| Une application de bureau distribuable à d'autres postes (autres anciens, pasteur, tablette) devient un objectif | Passer à B : Tauri 2 est stable, Wails v3 ne l'est pas encore |
| L'auteur veut un seul langage pour tout l'outillage local, RuVector compris | Passer à B |
| Le poste ne peut pas exécuter Edge en automatisation (politique de groupe, Edge désinstallé) | Passer à C ou à B |
| L'extraction depuis le PDF source doit être refaite régulièrement | Aucune des trois options ne le fait bien nativement ; conserver Python ou piloter `mutool` |

---

## 9. Esquisse de l'architecture cible (option A)

### 9.1 Commandes

| Commande | Rôle | Remplace |
|---|---|---|
| `coramdeo render <livrable.json> --out <dossier>` | Produit `.md`, `.html` et `.pdf` de même nom de base à partir du JSON des skills ; ne réécrit que ce dont les entrées ont changé | `python assets/generate-pdf.py` et l'adaptation manuelle du gabarit HTML |
| `coramdeo render-md <fichier.md>` | Rend un Markdown doctrinal en PDF maison | `md-to-pdf.py` |
| `coramdeo check [--fix-nfc]` | Liens des 925 pages (décodage percent, NFC, existence) ; structure de chaque dossier de livre selon l'anatomie du README ; UTF-8 sans BOM ; longueur des chemins ; plus tard, repérage des attributions à MacArthur non appuyées par le fichier `JMA` du livre | Passes d'agents du gauntlet |
| `coramdeo index` / `coramdeo search "<requête>"` | Construit et interroge un index FTS5 (`coramdeo.db`, ignoré par git) sur les `.md` ; recherche insensible aux accents, classement BM25, extraits | Rien |
| `coramdeo serve [--open]` | Sert le dépôt tel quel sur `localhost`, ajoute une page de recherche et un tableau de couverture par livre ; `--open` lance Edge en mode application | Ouverture des fichiers HTML un à un |
| `coramdeo mcp` | Expose `render`, `check`, `search` en outils MCP sur stdio pour Claude Code | L'appel Python dans les skills |

### 9.2 Flux de rendu

1. Le JSON du livrable est validé contre un schéma (champs obligatoires des skills : `source_level`, `pastor_name`, `sermon_links` facultatif).
2. Un modèle Go unique est construit ; le code du sermon Grace to You est déduit de l'URL, comme aujourd'hui dans `generate-pdf.py`.
3. Le gabarit HTML rend la présentation avec la feuille écran (thème sombre du site, identifiants de section conservés : `#contexte`, `#arriere-plan`, `#mots-cles`, `#commentateurs`, `#renvois`, `#themes`, `#reflexion`).
4. Le même HTML, avec la feuille impression (palette claire des PDF actuels, `lang="fr"`, `hyphens: auto`, polices embarquées), est imprimé par Edge via `Page.printToPDF`, en-tête et pied fournis par gabarit.
5. Le Markdown est écrit depuis le même modèle, avec les mêmes en-têtes que les fichiers existants (« Contexte du passage », « Apports des commentateurs », tableau à cinq colonnes des mots-clés).

### 9.3 Polices et déterminisme

Les polices OFL du dépôt (Crimson Pro, Arsenal SC) et, pour l'écran, Cormorant Garamond et EB Garamond, elles aussi sous OFL [probable], sont embarquées dans le binaire (`embed`) et servies au moteur de rendu par `@font-face` ; le PDF ne dépend plus des polices installées sur le poste, ce qui règle le cas des macrons de translittération que `pdf_utils.py` gérait en cherchant Times New Roman.

### 9.4 Particularités Windows à coder

- Localisation de `msedge.exe` par la clé `App Paths` du registre, avec repli sur le chemin d'installation habituel [probable].
- Page de code console 65001 au démarrage pour l'affichage des accents.
- Aucun traitement particulier des chemins longs : la bibliothèque standard s'en charge [confirmé]. `check` signale toutefois tout chemin qui dépasserait 240 caractères une fois absolu, par précaution.
- `check` refuse les noms en NFD, qu'un aller-retour par macOS pourrait introduire.

### 9.5 Contrat avec les skills

Dans chaque `SKILL.md`, l'étape « Exécuter `python assets/generate-pdf.py …` » devient « Exécuter `coramdeo render <json> --out "4 - Sermon/"` », et l'étape « Rédiger la présentation HTML en adaptant le gabarit » disparaît, puisque le HTML sort du même rendu. Le prérequis `pip install reportlab` disparaît. Le JSON ne change pas.

---

## 10. Feuille de route [hypothèse]

| Phase | Contenu | Critère de sortie | Ordre de grandeur |
|---|---|---|---|
| 0. Vérification et squelette | Test de `Page.printToPDF` sur l'Edge du poste ; squelette Go ; `check` des liens | Un PDF produit par Edge depuis une page existante ; rapport de liens sur les 925 pages | 1 séance |
| 1. `render` | Modèle, gabarits HTML écran et impression, Markdown ; comparaison côte à côte avec `Recherche-MacArthur-Jude-1-1-2.pdf` et `Serie-MacArthur-Psaume-119.pdf` produits par reportlab | Parité de contenu sur les deux livrables témoins ; bascule des deux `SKILL.md` | 2 à 3 séances |
| 2. `index`, `search`, `serve` | Index FTS5, page de recherche, tableau de couverture, mode `--open` | Recherche accentuée et non accentuée sur tout le corpus en moins d'une seconde | 2 séances |
| 3. `mcp` | Serveur stdio, trois outils, configuration dans Claude Code | Une recherche lancée sans Python ni shell depuis Claude Code | 1 séance |
| 4. Facultatif | `check` des attributions contre les fichiers `JMA` ; coque Tauri ou Wails si une condition du §8.3 se réalise | Selon le besoin | Hors plan |

---

## 11. Risques et vérifications préalables sur le poste

1. **Edge 141 et le mode headless.** Tester `Page.printToPDF` par `chromedp` avant tout ; sinon essayer `--headless=old` et `--user-data-dir` ; sinon, option C. [à vérifier]
2. **FTS5 dans `modernc.org/sqlite`.** Vérifier par `PRAGMA compile_options` au premier lancement. [à vérifier]
3. **Defender.** Compiler localement ; ajouter le dossier de compilation aux exclusions si un faux positif survient ; soumettre le fichier à Microsoft le cas échéant.
4. **Polices.** Confirmer que la licence OFL de Cormorant Garamond et d'EB Garamond autorise l'embarquement dans un exécutable (c'est le cas de l'OFL en général) [probable].
5. **Stabilité des PDF.** Confirmer que la régénération conditionnée au hachage supprime bien les diffs parasites sur quelques cycles ; à défaut, neutraliser la date de création par post-traitement.
6. **Windows sur ARM.** Si le portable est un ARM64, Go et `modernc.org/sqlite` couvrent `windows/arm64` [confirmé] ; Edge aussi.

---

## Annexe A. Sources vérifiées le 7 septembre 2026

- Go 1.27, notes de version et annonce (The Go Programming Language, août 2026) : <https://go.dev/doc/go1.27>, <https://go.dev/blog/go1.27>
- Rust 1.98.0 (Rust Blog, 20 août 2026) : <https://blog.rust-lang.org/2026/08/20/Rust-1.98.0/>
- Prérequis MSVC pour la cible `x86_64-pc-windows-msvc` (rustup, documentation) : <https://rust-lang.github.io/rustup/installation/windows-msvc.html>
- Tauri, versions de l'écosystème et versions de webview (Tauri, 2026) : <https://v2.tauri.app/release/>, <https://v2.tauri.app/reference/webview-versions/>
- Wails v3 bêta (Wails, blogue, 2026) et suivi bêta vers GA : <https://v3.wails.io/blog/wails-v3-beta/>, <https://github.com/wailsapp/wails/issues/5844>
- Typst 0.15.0 (15 juin 2026) et 0.15.1, caisse `typst` (Typst) : <https://github.com/typst/typst/releases/tag/v0.15.0>, <https://docs.rs/crate/typst/latest>, <https://typst.app/open-source/>
- SDK MCP officiel pour Go, versions (Model Context Protocol, avec Google) : <https://github.com/modelcontextprotocol/go-sdk/releases>
- SDK MCP officiel pour Rust, `rmcp` (Model Context Protocol) : <https://github.com/modelcontextprotocol/rust-sdk>
- Régression `--print-to-pdf` en mode headless depuis Edge 141 (Microsoft Q&A, 2025-2026) : <https://learn.microsoft.com/en-us/answers/questions/5579903/msedge-exe-does-not-save-html-file-as-pdf-in-comma>, <https://learn.microsoft.com/en-us/answers/questions/5729862/how-to-print-to-pdf-with-no-header-and-footer-when>
- WebView2 Evergreen préinstallé sur Windows 11 (Microsoft Learn) : <https://learn.microsoft.com/en-us/microsoft-edge/webview2/concepts/distribution>
- `modernc.org/sqlite` v1.58.0, SQLite 3.53.4, plateformes (pkg.go.dev, 1er septembre 2026) : <https://pkg.go.dev/modernc.org/sqlite>
- `go-pdf/fpdf` archivé sur GitHub, poursuivi sur Codeberg (dernier commit 27 mai 2026) : <https://github.com/go-pdf/fpdf>, <https://codeberg.org/go-pdf/fpdf>
- `maroto` v2 et sa dépendance à `gofpdf` (johnfercher/maroto, issue 431) : <https://github.com/johnfercher/maroto/issues/431>
- `signintech/gopdf` (MIT, justification, tableaux) : <https://github.com/signintech/gopdf>
- Césure CSS en français dans Chromium, y compris sous Windows (MDN browser-compat-data, issue 18987) : <https://github.com/mdn/browser-compat-data/issues/18987>
- Faux positifs Defender sur les binaires Go (microsoft/go, issue 1255) : <https://github.com/microsoft/go/issues/1255>
- Chemins longs dans `os` sous Windows (`fixLongPath`, `os/path_windows.go`) : <https://go.dev/src/os/path_windows.go>

## Annexe B. Ce qui n'a pas été vérifié

- Le comportement de `Page.printToPDF` (protocole DevTools) sur Edge 141 et suivants ; seul le drapeau de ligne de commande est documenté comme cassé.
- La présence de FTS5 dans la compilation de `modernc.org/sqlite` (fortement probable : des bibliothèques tierces bâtissent un index FTS5 typé par-dessus).
- La version et la stabilité d'API de `typst-as-lib` et de `typst-bake` (pages crates.io inaccessibles à l'outil de consultation).
- L'état de maintenance de `genpdf` (Rust).
- Les temps de rendu et les tailles de binaire avancés au §6 et au §5.2 : ordres de grandeur, non mesurés.
