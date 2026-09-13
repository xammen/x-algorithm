# Notes vérifiées — Algorithme « For You » de X

Source : dépôt `xai-org/x-algorithm` (fork local `x-algorithm/`, fork GitHub https://github.com/xammen/x-algorithm).
Tous les chemins sont relatifs à `x-algorithm/`. Valeurs vérifiées directement dans le code le 2026-09-13.

> Le dépôt ne contient **pas** les poids entraînés de production (checkpoints Phoenix), ni les prompts Grox, ni certaines règles anti-spam. Les valeurs par défaut des `param!` de `home-mixer/params/param.rs` sont documentées par le README comme reflétant les valeurs de production principales.

## 1. Formule de score

`home-mixer/scorers/ranking_scorer.rs`

```
score_pondéré = Σ ( poids_action × P(action | viewer, post) )
```

- Les poids multiplient des **probabilités prédites** par le modèle Phoenix, **pas** des compteurs bruts. (README + `param.rs:285-313`.)
- `offset_score()` (`ranking_scorer.rs:536-544`) :
  - net positif → `score + 0.001`
  - net négatif → `(score + negative_sum) / total_sum × 0.001` → écrasé dans une bande ~[0, 0.001)
  - Conséquence : **tout post net-positif bat tout post net-négatif** ; un signal négatif prédit fait chuter le post au plancher.
- Ajustements appliqués ensuite (`score()`): boost cold-start → décote de diversité d'auteur → décote out-of-network (OON).

## 2. Poids des actions (`home-mixer/params/param.rs:314-480`)

| Action | Param | Poids |
|---|---|---|
| Share via copy link | `ShareViaCopyLinkWeight` | **20.0** |
| Reply | `ReplyWeight` | 5.0 |
| + boost reply si follow mutuel | `BidirectionalFollowReplyWeightBoost` | **+15.0** (→ 20.0) |
| Quote | `QuoteWeight` | 5.0 |
| Share via DM | `ShareViaDmWeight` | 5.0 |
| Follow author | `FollowAuthorWeight` | 4.0 |
| Share | `ShareWeight` | 2.0 |
| Repost/Retweet | `RetweetWeight` | 1.0 |
| Like | `FavoriteWeight` | 0.5 |
| Click | `ClickWeight` | 0.4 |
| Open link | `OpenLinkWeight` | 0.2 |
| Video open | `VideoOpenWeight` | 0.07 |
| Dwell (binaire) | `DwellWeight` | 0.05 |
| Dwell time (continu, /unité) | `ContDwellTimeWeight` | 0.004 |
| Photo expand | `PhotoExpandWeight` | 0.05 |
| Quoted click | `QuotedClickWeight` | 0.05 |
| Post unexplored | `PostUnexploredWeight` | 0.02 (in-network only) |
| Profile click | `ProfileClickWeight` | **0.0** |
| VQV / quoted VQV | `VqvWeight` / `QuotedVqvWeight` | **0.0** |
| Not dwelled | `NotDwelledWeight` | -0.02 |
| Not interested | `NotInterestedWeight` | **-43.2** |
| Block author | `BlockAuthorWeight` | **-31.2** |
| Mute author | `MuteAuthorWeight` | **-58.8** |
| Report | `ReportWeight` | **-234.0** |

**Il n'y a pas de poids pour les bookmarks** dans ce scoring. Les bookmarks ne comptent donc pas directement — mais ils corrèlent avec le « copy link share » (poids 20).

Ajustements de score :
- `OonWeightFactor = 0.75` (`param.rs:254`) : ×0.75 pour tout contenu out-of-network, **et aussi** pour les réponses/reposts in-network.
- `TopicOonWeightFactor = 0.5` pour les requêtes par thème.
- Diversité d'auteur : `AuthorDiversityDecay = 0.5`, `AuthorDiversityFloor = 0.25` →
  - 1er post d'un auteur : ×1.0
  - 2e : ×0.625
  - 3e : ×0.4375
  - 4e : ×0.34375 … plancher ×0.25
  (calcul : `(1-0.25) × 0.5^k + 0.25`, `ranking_scorer.rs:625-627`)

## 3. Constantes (`home-mixer/params/config.rs`)

| Constante | Valeur |
|---|---|
| `TOP_K_CANDIDATES_TO_SELECT` | 50 |
| `RESULT_SIZE` | 35 (+ 4 modules + 8 frames = `FOR_YOU_MAX_RESULT_SIZE` 47) |
| `MAX_POST_AGE` | **48 h** |
| `NEGATIVE_SCORES_OFFSET` | 0.001 |
| `NEW_USER_OON_WEIGHT_FACTOR` | 0.00001 (branche inatteignable par défaut, `NewUserAgeThresholdSecs=0`) |
| `UAS_WINDOW_TIME_MS` | 300 000 (fenêtre d'historique d'actions : 5 min) |

## 4. Filtres (`home-mixer/filters/*.rs`)

Retirés avant scoring : doublons, âge > 48 h, tes propres posts, **réponses ET reposts out-of-network** (`oon_retweet_reply_filter.rs`), réponses sans parent, reposts dupliqués, contenu abonné non souscrit, posts déjà vus/servis, mots-clés masqués, blocages/mutes, filtre Brésil 2026.
Après sélection : `VFFilter` (drop si `visibility_action = Drop/Tombstone/NotEvaluated`), `AncillaryVFFilter`, `DedupConversationFilter` (1 post par conversation).
Désactivés par défaut : `new_user_min_engagement_filter`, `inventory_holdout`.

Point clé : **une réponse ou un repost ne peut PAS être découvert out-of-network.** Seuls les originaux sont découverts par des gens qui ne te suivent pas.

## 5. Récupération des candidats (sources)

- **Thunder** (in-network) : posts des comptes suivis, rétention **48 h**, max 1200, tri **strictement par récence**. Caps/auteur : 50 originaux, 30 réponses, 100 vidéos.
- **Phoenix** (OON) : recherche two-tower, max 1000. L'utilisateur est représenté par **son historique d'engagement** (pas d'embedding d'ID utilisateur). Les **likes** sont le signal positif contrastif d'entraînement de la récupération.
- **SimClusters** (OON) : max 800, graines = signaux d'engagement explicites/implicites de l'utilisateur, ANN de similarité de co-engagement.
- Seuls **50** candidats retenus, **35** affichés par requête.

## 6. Cold start / petits comptes (`home-mixer/scorers/author_cold_start.rs`, `param.rs:658-737`)

Actif par défaut (`EnableViewerColdStart=true`). Un **original** éligible est propulsé au score du ~15e meilleur candidat.
Conditions d'éligibilité :
- post **original** (pas une réponse, pas un repost) ;
- auteur ≤ `ColdStartFollowerCap = 1000` followers ;
- post < `ColdStartMaxPostAgeSecs = 86400` (24 h) ;
- `view_count_on_home < ColdStartImpressionThreshold = 1000` ;
- doit être classé dans les `LowImpressionsMaxPositionRatio = 0.85` (top 85 %) des candidats à score non nul.
- Slot cible `ColdStartSlotMin=15` … `SlotMax=16`. **Un seul post boosté par requête.**

## 7. Visibilité (réduit la portée hors-followers)

`visibility-filtering/` et `under-the-hood/strato/lib/underTheHoodLabels.strato` :
- Label `SPAM` → **le post n'est pas montré du tout**.
- `SPAM_HIGH_RECALL`, `MALICIOUS_URL`, `DO_NOT_AMPLIFY`, `FOSNR_ABUSE_INSULTS`, `NSFW_*`, `GORE_AND_VIOLENCE_HIGH_PRECISION`, labels de compte (`SpamHighRecall`, `AbusiveHighRecall`, `DoNotAmplify`, `Nsfw*`…) → **masqué des recommandations pour les non-followers**.
- Posts « stale » : TTL 14 j côté features Phoenix (mais l'`AgeFilter` 48 h est plus strict).
- Outil de transparence : https://x.com/i/under_the_hood

## 8. Ce que le dépôt ne contient pas

Poids entraînés de production (Phoenix), prompts Grox, seuils BDSM, une partie des règles botmaker, et le crate externe `candidate-pipeline`. Les conclusions restent directionnelles : les expériences peuvent faire varier les valeurs sur une fraction du trafic.
