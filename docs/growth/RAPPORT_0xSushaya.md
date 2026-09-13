# Comment percer sur X — Audit de @0xSushaya à partir de l'algorithme open-source de X

> Analyse du 13 septembre 2026 · Source : code `xai-org/x-algorithm` (forké sur https://github.com/xammen/x-algorithm) + données réelles du compte via AnyAPI.
> Les poids et filtres cités sont vérifiés dans le code. Les checkpoints du modèle de production ne sont pas publiés, donc les conclusions sont **directionnelles** (voir Limites).

---

## 1. Synthèse (TL;DR)

Ton problème n'est pas « l'algo te pénalise ». C'est que **tu produis très peu de ce que l'algo distribue, et tes signaux d'amplification sont quasi nuls**.

Les 5 vérités qui expliquent tes vues faibles :

1. **Ton réservoir in-network est minuscule** : 138 followers. Même à 100 % de portée auprès d'eux, ça fait 138 vues. Tout le reste doit venir du hors-réseau (OON).
2. **L'OON ne distribue que des posts originaux.** Les réponses et reposts sont *supprimés* avant scoring pour les non-followers. Or **77 % de ton activité récente = des réponses**.
3. **Ton taux d'amplification est quasi nul** : ~0,01 % de retweets. L'algo pondère reply ×5, quote ×5, share ×2, copie-lien ×20, follow ×4, alors qu'un **like ne vaut que ×0,5**. Tu collectionnes des likes, pas ce qui compte.
4. **Tu n'as presque pas d'historique d'engagement exploitable**, donc la récupération hors-réseau (Phoenix/SimClusters), qui s'appuie sur ton historique, ne t'associe à personne.
5. **Tu mélanges FR et EN, et tu postes souvent des liens nus** : ça dilue ton ciblage et tes embeddings, et les liens sont le format le moins poussé.

Bonne nouvelle : tu es **exactement le profil que le « cold-start boost » de X est fait pour aider** (≤ 1000 followers, chaque original frais < 24 h peut être propulsé au niveau du ~15e meilleur candidat). C'est un avantage qui disparaît au-delà de 1000 followers. **Il faut l'exploiter maintenant, à fond.**

---

## 2. Ce que j'ai fait

- Forké + cloné le code source de l'algo X : https://github.com/xammen/x-algorithm
- Extrait 83 tweets/réponses + 40 éléments de ton onglet Posts (AnyAPI : `twitter.profile`, `twitter.user_tweets`, `twitter.user_posts`, `twitter.followers`)
- Intégré ton export officiel **Under the Hood** (labels de juillet 2026)
- Épluché le scoring (`home-mixer/params/param.rs`, `ranking_scorer.rs`), les filtres (`home-mixer/filters/*`), la récupération (`thunder/`, `phoenix/`, `simclusters`), la visibilité (`visibility-filtering/`) et le cold-start (`author_cold_start.rs`)
- Calculé tes statistiques réelles et croisé avec les leviers du code

Artefacts :
- `docs/ALGORITHM_NOTES.md` — la mécanique exacte de l'algo, vérifiée
- `data/*.json` — ton corpus brut
- `scripts/analyze.py`, `scripts/deep_analysis.py` — l'analyse reproductible

---

## 3. Ton compte en chiffres (snapshot 13/09/2026)

| | @0xSushaya |
|---|---|
| Nom affiché | xam |
| Followers / Following | **138** / 133 |
| Tweets | 536 |
| Vérifié (bleu) | oui |
| Bio | « solo nerd \| life, tech, dev, security , ia » |
| Site | hiii.boo |

### Activité et portée (échantillon analysé)

| Métrique | Valeur |
|---|---|
| Originaux vs réponses (fenêtre longue, n=83) | **19 originaux / 64 réponses (77 % de réponses)** |
| Onglet Posts, 72 derniers jours | 21 originaux + **19 reposts** |
| Vues médianes — originaux récents (72 j) | **≈ 73 vues** |
| Vues médianes — originaux (fenêtre longue) | 421 |
| Vues médianes — réponses | 699 (jusqu'à 19 989) |
| Taux de like | ~4 pour 1 000 vues (0,4 %) |
| **Taux de retweet** | **~0,1 pour 1 000 vues (0,01 %)** |
| Taux de bookmark | ~1,8 pour 1 000 vues |
| Langues | 51 FR · 26 EN (mélange) |
| Posts avec lien externe | 18/83 |
| Meilleur post | Fuite CAF (18/12/2025) : **9 821 vues, 31 likes, 6 RT, 34 bookmarks, 10 réponses** |

### Ton réseau (136 followers échantillonnés)

- Médiane followers de tes followers : **102** · 78/136 ont < 200 followers
- **10 comptes > 10 k followers** : @Teknium (126 k, Nous Research), @darylginn (123 k), @seblatombe (109 k, cyber/OSINT), @Sylvqin (58 k), @_3emeOeil (52 k), @oliverhenry (51 k), @zacjohnson (48 k), @adamludwin (23 k), @PetioRolecks (12 k), @LottoLabs (11 k)
- 0/136 vérifiés

**Lecture :** tu as une poignée d'amplificateurs potentiels sérieux dans ton audience, mais la masse est constituée de petits comptes. La cible la plus rentable = ces 10 gros comptes, qu'il faut transformer en **follows mutuels** (voir le bonus reply +15).

### Tes meilleures réponses (par vues)

| Cible | Vues | Likes |
|---|---|---|
| @siliconcarnesf | 19 989 | 42 |
| @RayaneRachid_ (×3) | 11 065 | 23 |
| @Simon_Hypixel (×2) | 10 234 | 28 |
| @Tur24Tur | 6 022 | 78 |
| @om_patel5 | 3 418 | 86 |

→ Ta stratégie « reply guy » **capte des vues** mais **ne convertit presque pas** en followers (aucune de ces réponses ne dépasse ~1 follow visible, et ton total grimpe très lentement).

### Under the Hood (rapport X de juillet 2026)

Export officiel : période **01→31 juillet 2026**, **47 posts analysés**.

| | Nombre |
|---|---|
| Labels de compte (spam, abuse, NSFW, `DoNotAmplify`…) | **0** |
| Labels de post (`SPAM`, `NSFW_*`, `MALICIOUS_URL`, `FDO_NOT_AMPLIFY`…) | **0** |

✅ **Conclusion majeure : ton compte est 100 % propre.** Aucun label ne limite ta visibilité. Ton manque de vues **n'est pas un problème d'enforcement / de spam-flag** — c'est uniquement un problème de **contenu, de format et d'amplification**. C'est une excellente nouvelle : ça se corrige.

⚠️ Note : ce rapport est **mensuel** (généré le 08/08). Reprends-en un pour août/septembre après avoir appliqué le plan.

---

## 4. Comment marche l'algo (l'essentiel)

1. **Deux viviers** : in-network (`thunder/`, posts des 48 h des comptes que tu suis, triés par récence) + out-of-network (`phoenix/` + `simclusters`).
2. **Filtres** : posts > **48 h** jetés ; tes propres posts exclus ; **réponses/reposts OON jetés** ; doublons, déjà-vus/déjà-servis jetés ; labels de spam/NSFW → invisibles aux non-followers.
3. **Score** = `Σ poids × P(action prédite)`, puis décote de diversité d'auteur, puis ×0,75 si hors-réseau.
4. **Sélection** : top 50 → 35 posts affichés par requête.
5. **Petits comptes** : original < 24 h, auteur ≤ 1000 followers, < 1000 impressions → propulsé au score du ~15e candidat (1 post/requête).

### Table des poids (extrait vérifié)

| Action | Poids | | Action | Poids |
|---|--:|---|---|--:|
| Copie du lien | **20,0** | | Repost | 1,0 |
| Réponse (follow mutuel) | **20,0** | | **Like** | **0,5** |
| Réponse | 5,0 | | Clic | 0,4 |
| Quote | 5,0 | | Ouvrir lien | 0,2 |
| Partage en DM | 5,0 | | Expand photo | 0,05 |
| **Follow auteur** | **4,0** | | Dwell (lecture) | 0,05 |
| Partage | 2,0 | | Clic profil / VQV | **0,0** |

Négatifs : Report **-234** · Mute **-58,8** · Not interested **-43,2** · Block **-31,2**.
→ Un seul signal négatif prédit écrase plusieurs dizaines de likes.

---

## 5. Diagnostic : pourquoi tu fais peu de vues

| # | Cause (dans le code) | Ton cas |
|---|---|---|
| 1 | **Base in-network minuscule** (Thunder = followers × 48 h) | 138 followers → plafond théorique ~138 vues/original |
| 2 | **L'OON ne récupère que des originaux** (`oon_retweet_reply_filter`) | 77 % de ton activité = réponses → invisible hors-réseau |
| 3 | **Trop peu d'originaux frais** : seuls les posts < 48 h concourent, et le cold-start exige < 24 h | 47 posts en juillet mais seulement ~1 original tous les 3 jours → souvent rien de frais à propulser |
| 4 | **Amplification ~0** : retweet ×1, quote ×5, copie-lien ×20 | 0,01 % de retweets → ton score reste au plancher |
| 5 | **Pas d'historique d'engagement exploitable** pour la récupération OON (Phoenix/SimClusters partent de ton historique) | peu de likes/replies → tu n'es récupéré pour personne d'autre |
| 6 | **Mélange FR/EN** (51/26) | embeddings flous → similarité utilisateurs/audience faible |
| 7 | **Posts-lien nus** : open-link ×0,2 + risque de label spam | médiane 421 vues avec lien vs 1 478 pour le texte long |
| 8 | **Décote de diversité** : ×0,625 dès le 2e post, ×0,4375 au 3e | flooder ne sert à rien, il faut étaler |
| 9 | **Cold-start non exploité** : tu es éligible mais il faut poster frais, original, régulièrement | tu publies trop peu pour en profiter |
| 10 | **Décote OON ×0,75** tant qu'on ne te suit pas | convertir un viewer en follow = **+33 % de score** |
| 11 | ~~Labels de visibilité~~ | ✅ **Vérifié : 0 label** (Under the Hood, juillet 2026) → ton problème n'est **pas** l'enforcement |

**Le cercle vicieux :** peu de followers → peu d'in-network → peu d'engagement → pas d'historique → pas de découverte OON → peu de nouveaux followers. **Le moyen de casser la boucle = les originaux frais + le cold-start + les follows mutuels + le contenu « partageable ».**

**Le contre-exemple dans tes données :** ton post « Fuite CAF » (original, actu, + un outil, 34 bookmarks) a fait **9 821 vues**. C'est **le template gagnant**. Le problème, c'est que tu ne l'as pas répété.

---

## 6. Plan d'action

### Phase 0 — Audit (aujourd'hui, 30 min)
- [x] ✅ **Labels vérifiés** (Under the Hood, juillet 2026) : **0 label compte, 0 label post**. Rien à nettoyer côté enforcement.
- [ ] Vérifie qu'aucun de tes posts récents ne contient de lien raccourci douteux (`t.co` vers des proxys, etc.).
- [ ] Supprime les 19 reposts récents de l'onglet : ils ne t'apportent **aucune** portée OON (reposts jetés) et diluent ton profil.

### Phase 1 — Positionnement (48 h)
- [ ] **Choisis UNE niche + UNE langue.** Recommandation : **anglais** (audience globale, tech/IA/cyber) OU **français** (cyber/OSINT où @seblatombe est un hub). Pas les deux.
- [ ] Réécris ta bio avec une promesse claire, pas « life, tech, dev, security, ia » :
  > ex. « Je construis des outils qui traquent les fuites de données. J'écris sur la cyber et l'IA. → hiii.boo »
- [ ] Épingle ton meilleur post (ou un thread « start here » qui explique qui tu es et ce que tu publies).
- [ ] **Follow-back et engage les 10 gros comptes** (Teknium, seblatombe, darylginn, Sylvqin, _3emeOeil, LottoLabs, oliverhenry, zacjohnson, adamludwin, PetioRolecks). Un follow mutuel = **+15 de poids sur les réponses à leurs posts** (×3 par rapport à un reply normal).

### Phase 2 — Moteur de contenu (routine 30 jours)
Objectif : **2-3 originaux/jour, espacés de 2-4 h** (la décote de diversité punit le flood).
- [ ] **Poste des originaux, pas des réponses** (ratio cible ≥ 50/50). Les réponses restent utiles, mais comme *canal d'acquisition*, pas comme cœur d'activité.
- [ ] **Supprime les liens externes du corps du post.** Mets le lien en 2e tweet ou en réponse. `open_link` ne vaut que 0,2 et un lien dans le post te coûte de la portée.
- [ ] **Formats qui marchent chez toi** : texte long (médiane 1 478 vues), image/screenshot (1 268), threads. Évite le texte nu court (192).
- [ ] **Hook en 1re ligne** : les premières 1-2 secondes décident du dwell. Pas de « bonjour », pas de contexte, direct la valeur/le scoop.
- [ ] **Vise les partages, pas les likes.** Les 3 formats de partage les plus valorisés (copie-lien ×20, DM ×5, quote ×5) = contenus **utiles, copiables, mémorables** :
  - listes / checklists / ressources (« les 8 outils que j'utilise pour X »),
  - scoops / analyses exclusives (ton template CAF),
  - memes / prises nettes (partage DM),
  - prises de position qui invitent au quote.
- [ ] **Pose des questions / provoque la réponse.** reply ×5 vs like ×0,5.
- [ ] **Un post = une idée.** Recycle : un scoop → 1 thread + 3 posts atomiques + 1 réponse longue ailleurs.
- [ ] **Poste aux heures chaudes** d'après tes données (mar/mer, ~00-02 h, 06 h, 09-10 h, 14 h UTC) — à affiner avec les stats de ton X Analytics.

### Phase 3 — Amplification (continu)
- [ ] **Reply stratégie ciblée** : sois dans les **15-30 premières minutes** sous les posts de tes gros followers/mutuals, avec des réponses **à valeur** (pas « gg »). Tu captes des vues + des follows.
- [ ] **Construis 5-10 mutuals actifs** dans ta niche : follow-back + réponses régulières. Chaque mutual débloque le bonus reply +15 sur tes réponses à leurs originaux.
- [ ] **1 « scoop/outil » par semaine** : c'est ton format le plus fort (CAF = 9 821 vues). Sources : fuites, releases, benchmarks, veille.
- [ ] **Demande le follow explicitement mais élégamment** en fin de post à forte valeur (« follow for more X ») — `FollowAuthorWeight = 4.0`.
- [ ] **Ne supprime pas / ne reposte pas en boucle** : les filtres « déjà vu/déjà servi » + dédup empêchent de re-toucher les mêmes personnes.

### Phase 4 — Mesure et itération (hebdo)
- [ ] Note chaque semaine : followers, vues médianes des originaux, **taux de retweet/quote/copie-lien**, meilleur post.
- [ ] Chaque dimanche : identifie le top 3 des posts, extrais la structure, **refais-la**.
- [ ] Coupe ce qui ne dépasse pas la médiane 3 fois de suite.

---

## 7. 12 templates de posts (prêts à remplir)

1. **Scoop outillé** : « [Actu] vient de se produire. J'ai récupéré les données : [chiffre choc]. Voici ce que personne ne dit. »
2. **Outil original** : « J'ai construit [outil] en [durée]. Il fait X. Utilise-le gratuitement ici → (lien en réponse). »
3. **Liste copiable** : « 10 outils que j'utilise tous les jours pour [tâche]. Sauvegarde ce post. 🧵 »
4. **Démythification** : « Tout le monde dit [idée reçue]. C'est faux, et voici pourquoi (données à l'appui). »
5. **Avant/après** : « Il y a 6 mois : [état]. Aujourd'hui : [état]. Le seul truc qui a changé : [leçon]. »
6. **Teardown** : « J'ai analysé [produit/fuite]. 3 choses que vous ratez. » + screenshot
7. **Question qui clive (sans ragebait)** : « Question sincère : pourquoi [X] ? Je lis vos réponses. »
8. **Prédiction** : « Je pense que [X] va se passer d'ici [échéance]. Voici mon raisonnement. Quote ce post dans 3 mois. »
9. **Coulisses** : « Ce que personne ne montre sur [métier/projet]. Screenshot à l'appui. »
10. **Comparatif** : « [A] vs [B] : j'ai testé les deux pendant [durée]. Tableau à l'appui. »
11. **Ressource** : « J'ai compilé [N] ressources sur [sujet]. C'est gratuit, copie le lien. »
12. **Réaction rapide** : dans l'heure, sous un gros post : « Le vrai sujet que personne ne relève, c'est [angle]. »

---

## 8. Routine quotidienne recommandée

| Quand | Action | Temps |
|---|---|---|
| Matin | 1 original (scoop/analyse/outil) | 20 min |
| Midi | 3-5 réponses à valeur sur gros comptes/mutuals | 15 min |
| Après-midi | 1 original (liste/meme/prise) | 15 min |
| Soir | 1 original + réponses aux commentaires | 20 min |
| Dimanche | Revue des stats + recyclage des meilleurs formats | 30 min |

---

## 9. KPIs à suivre

- Followers (net/semaine)
- **Vues médianes des originaux** (ton KPI n°1)
- **Taux de retweet + quote + copie-lien** (le cœur du score)
- Nombre de mutuals actifs dans ta niche
- % d'originaux dans ton activité (cible ≥ 50 %)
- Aucun label de visibilité sur « Under the Hood »

---

## 10. Limites de l'analyse

- Les **poids entraînés de production** de Phoenix ne sont pas publiés : ce rapport s'appuie sur les paramètres par défaut du code, que X documente comme reflétant la production, plus des expériences sur une partie du trafic.
- Le détail de certaines règles anti-spam (botmaker) et les seuils d'enforcement sont volontairement absents.
- Le corpus est limité par l'API (~83 tweets récents) : ce sont tes données récentes, pas l'historique complet des 536 tweets. Le rapport Under the Hood confirme que tu as publié **47 posts en juillet**, donc l'échantillon API sous-estime ton volume réel (mais il surestime la part d'originaux).
- `twitter.following` a échoué côté fournisseur (bug API, non bloquant) : l'analyse des mutuals repose sur l'échantillon de followers.

---

*Rapport généré le 13/09/2026. Code source : https://github.com/xammen/x-algorithm · Mécanique détaillée : `docs/ALGORITHM_NOTES.md`.*
