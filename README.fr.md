[English](README.md) · **Français**

# Rig Open-Air 4x RTX 5090 pour l'Inférence LLM Locale

> Le parcours d'un cinéaste construisant une bête à 128 Go de VRAM pour l'inférence et le fine-tuning d'IA en local, avec un budget prosumer.

## Pourquoi ce Build

Je suis cinéaste, je travaille avec l'inférence IA locale et la recherche créative. Les coûts de GPU cloud s'accumulent vite, et j'avais besoin d'une machine capable de :

- Faire tourner des **modèles 70B+ paramètres** quantifiés (GGUF Q4/Q5) à vitesse interactive, comme serveur d'inférence agentique permanent
- Faire tourner des **workflows ComfyUI** pour Flux.2 et LTX Video 2 (qui nécessite jusqu'à 27 Go de VRAM pour son seul encodeur de texte)
- Servir de backend local pour **Claude Code et les pipelines agentiques** (Hermes Agent, modèles open-source Tier 2/3), toujours allumé, sans file d'attente, sans coût d'API
- Gérer de l'**inférence multi-modèles** via vLLM ou llama.cpp avec parallélisme tensoriel sur les 4 GPU
- Servir tout le **réseau Tailscale** comme un nœud API privé LLM/image/vidéo

La réponse : **4x RTX 5090** (128 Go de VRAM au total) sur un châssis open-air, construit progressivement avec des pièces déjà en ma possession issues de builds précédents et un peu de sourcing malin.

### L'Architecture de Modèles Derrière ce Build

Faire tourner des modèles frontier en local est en grande partie un faux problème, Claude 4, GPT-5 et Gemini Ultra ne sont pas open-source et ne tourneront sur aucun hardware grand public. Le bon modèle mental est une stack à trois niveaux :

```
Tier 1, API frontier (Claude / GPT / Gemini)
  → Raisonnement complexe, tâches créatives ponctuelles, pipelines de production
  → Routé via LiteLLM, même endpoint, on change de modèle en une ligne de config

Tier 2, Open-source local équivalent frontier (70B+ dense, 200B+ MoE en Q4-Q5)
  → Ce rig. Tâches agentiques quotidiennes, workflows Hermes Agent, requêtes de recherche
  → Coût API nul, pas de rate limit, aucune donnée ne sort du réseau, toujours chaud

Tier 3, Petits modèles locaux distillés (7-32B, turbo-quant)
  → Agents en arrière-plan, extraction structurée, classification, résumé
  → Tourne sur un seul GPU, les autres cartes restent libres pour le Tier 2 ou ComfyUI en simultané
```

**Pourquoi ne pas simplement utiliser des API pour tout ?** Au volume de requêtes des pipelines agentiques quotidiens (des centaines de requêtes/jour pour la recherche, l'analyse de plans, le traitement de documents), les coûts d'API s'accumulent vite. Le seuil de rentabilité du rig face au tarif des API est atteint en moins de 12 mois d'usage régulier. Après ça, chaque requête ne coûte que de l'électricité (~0,001 €/requête à 0,22 €/kWh).

**LiteLLM comme colle :** un unique proxy LiteLLM local expose tous les niveaux sous un seul endpoint compatible OpenAI. Hermes Agent, Claude Code, ou tout autre framework envoie ses requêtes à `http://localhost:4000` sans savoir, ni s'en soucier, si la réponse vient d'un Qwen3 local ou d'une API Claude distante.

### Rien ne Remplace Rien : la Chaîne de Fallback Complète

Ces niveaux sont **complémentaires, pas substituables**. Quand une couche tombe, la suivante rattrape. Le rig 4x5090 est la dernière couche, celle qui ne tombe jamais.

```
Claude Code / API Anthropic
  ↓ si down ou rate-limited
AWS Bedrock (même modèle Claude, infrastructure différente)
  ↓ si Bedrock est aussi indisponible
API DeepSeek V4 (ou OpenRouter → n'importe quel modèle frontier)
  ↓ si toutes les API commerciales échouent ou si le coût compte
Modèle open-source complet en local, Tier 2 (modèle 70B+, vérifier le leaderboard, ce rig)
  ↓ si le modèle complet est occupé ou si la tâche est routinière
Modèle distillé en local, Tier 3 (7-14B turbo-quant, un seul GPU)
```

**LiteLLM gère ça automatiquement** via le routage `fallbacks` et `model_list` :

```yaml
# litellm_config.yaml
model_list:
  - model_name: primary
    litellm_params:
      model: anthropic/claude-opus-4-6
      fallbacks: [bedrock/claude-opus-4-6, deepseek/deepseek-chat, ollama/qwen3:72b]

  - model_name: agent-daily  # tâches Hermes Agent en arrière-plan
    litellm_params:
      model: ollama/qwen3:32b  # Tier 2 local
      fallbacks: [deepseek/deepseek-chat]

  - model_name: agent-fast   # extraction/classification routinière
    litellm_params:
      model: ollama/qwen3:8b  # Tier 3 local, turbo-quant
```

Résultat : n'importe quel framework d'agent qui tape sur `http://localhost:4000` obtient un failover automatique sans changement de code. Claude Code down ? Invisible. Anthropic et Bedrock down en même temps ? Le rig répond à la même requête en local. **Le hardware local est le seul nœud à 100% de disponibilité dans cette chaîne.**

### Stratégie de Routage des Tokens : Coût, Latence, Résilience

Ne jamais router tout le trafic vers un seul fournisseur. La stratégie optimale répartit la charge selon la complexité de la tâche, le volume, et les exigences de latence :

| Type de tâche | Routé vers | Pourquoi |
|-----------|----------|-----|
| Décision créative complexe unique | API Claude (Tier 1) | Nécessite du raisonnement frontier, faible volume |
| Synthèse de recherche, analyse de documents | Tier 2 local (70B+) | Volume élevé, aucune donnée ne sort de la machine |
| Appels d'outils par un agent, extraction structurée | Tier 3 local (7-14B turbo-quant) | 100-200 tok/s, coût marginal de 0 € |
| Génération de code (chemin critique) | API Claude → fallback local | La qualité compte, fallback si l'API est down |
| Indexation en arrière-plan, classification | Local uniquement (Tier 3) | Volume élevé, routinier, pas besoin d'internet |
| Contenu sensible côté confidentialité | Selon le contexte | Dépend du type de donnée, de la juridiction et du workflow, le local pour un contrôle maximal, mais pas une règle absolue |

**Principe offline-first :** tout workflow d'agent qui peut tourner en local devrait tourner en local. Internet tombe, le rig continue de travailler. Un rate limit d'API est atteint, le rig absorbe le trop-plein. Le cloud est le chemin d'amélioration, pas la dépendance.

**Répartition pratique des tokens (exemple de pipeline quotidien) :**
- ~10% des requêtes vont au Tier 1 (API frontier), les coûteuses, les irremplaçables
- ~40% vont au Tier 2 (70B+ local), qualité proche du frontier, coût d'API nul
- ~50% vont au Tier 3 (distillé local), rapide, pas cher, tâches en arrière-plan

Résultat : les coûts d'API chutent de 80-90% par rapport à tout router vers Claude/GPT, tout en gardant la qualité frontier pour les tâches qui en ont réellement besoin.

> **Sur la confidentialité :** ce qui est « sensible » dépend du contexte, ce qui compte, c'est le type de donnée, la juridiction, et qui doit y avoir accès. Le local donne un contrôle maximal. L'API donne une capacité maximale. La bonne réponse change selon le projet, le client, le pays. Rester ouvert aux deux, et laisser la config de routage faire office de politique.

---

## Liste Complète des Pièces

| Composant | Modèle | Qté | Prix unitaire (TTC) | Total (TTC) |
|-----------|-------|-----|-------------------|-------------|
| **CPU** | AMD Ryzen 9 9950X (4,3 / 5,7 GHz) | 1 | 649,95 € | 649,95 € |
| **Carte mère** | ASUS ProArt X870E-CREATOR WIFI | 1 | 549,95 € | 549,95 € |
| **RAM** | G.Skill Flare X5 Low Profile 96 Go (2×48 Go) DDR5-6000 CL30 | 2 | 1 299,95 € | 2 599,90 € |
| **GPU** | MSI GeForce RTX 5090 32G VANGUARD SOC | 4 | ~4 100 €* | ~16 400 €* |
| **Stockage** | Samsung 9100 PRO 8 To M.2 NVMe PCIe 5.0 | 1 | 1 249,95 € | 1 249,95 € |
| **Ventirad CPU** | Noctua NH-D15 Chromax Black | 1 | 149,95 € | 149,95 € |
| **AIO (de secours)** | Cooler Master MasterLiquid 240 Core II ARGB | 1 | 79,95 € | 79,95 € |
| **Ventilateur** | Noctua NF-A12x25 PWM chromax.black.swap | 1 | 49,96 € | 49,96 € |
| **Ventilateur** | Noctua NF-A12x15 PWM chromax.black.swap | 1 | 34,96 € | 34,96 € |
| **Alim (ATX)** | Corsair SF1000 SFX 80+ Platinum | 1 | 214,95 € | 214,95 € |
| **Alim (serveur)** | HP 1200W Server PSU (80+ Platinum) | 2 | ~40 € (occasion) | ~80 € |
| **Alim (pieuvre)** | Alim breakout chinoise 1600W (AliExpress) | 1 | ~60 € | ~60 € |
| **Châssis** | Châssis open-air mining/GPU (6-8 GPU) | 1 | ~120 € | ~120 € |
| **Risers** | Risers PCIe 5.0 (LINKUP AVA5 ou équivalent) | 4 | ~70 € | ~280 € |
| **Ventilateurs (extra)** | Arctic P12/P14 PWM (pour l'aération des GPU) | 6 | ~10 € | ~60 € |
| **Cartes breakout** | Breakout alim HP + câbles 16 broches | 2 | ~35 € | ~70 € |

**\*Prix de rue des GPU en avril 2026 (idealo.fr ~4 000-4 300 € ; le MSRP était de 2 699 € au lancement en novembre 2025, mais l'offre n'a pas rattrapé la demande). Vérifier [idealo.fr](https://www.idealo.fr/prix/205775927/msi-geforce-rtx-5090.html) pour le prix actuel.**

### Coût Total Estimé

| Catégorie | Coût |
|----------|------|
| Composants (neufs, TTC) | ~21 180 € |
| Pièces additionnelles (risers, ventilateurs, câbles) | ~410 € |
| **Total** | **~21 590 €** |

*Les prix des GPU sont volatils, la RTX 5090 a été lancée à ~2 699 € en novembre 2025 et est montée à 4 000-4 300 € de prix de rue en raison d'une demande soutenue. Les prix peuvent bouger significativement dans les 6-12 prochains mois.*

Posséder déjà les alims serveur, le châssis open-air et l'alim pieuvre a fait économiser ~260 €.

---

## Plongée dans l'Architecture

### Le Problème des Voies PCIe sur AM5

C'est la chose la plus importante à comprendre sur ce build, et ce que la plupart des guides survolent.

**L'AM5 (Ryzen série 9000) fournit 28 voies PCIe 5.0 depuis le CPU :**

| Allocation | Voies | Génération |
|------------|-------|------------|
| Slots GPU (PCIEX16_1 + _2) | 16 | PCIe 5.0 |
| M.2 NVMe (2 slots) | 8 | PCIe 5.0 |
| Uplink chipset | 4 | PCIe 5.0 |

Le **chipset X870E** ajoute ensuite 36-44 voies PCIe 4.0 pour les slots d'extension secondaires, du M.2 supplémentaire, l'USB, le SATA, etc. Mais le CPU lui-même ne fournit que **16 voies** pour les slots GPU au total.

**Ce que propose réellement l'ASUS ProArt X870E-CREATOR WIFI :**

- **Slot 1 (PCIEX16_1) :** PCIe 5.0 x16 depuis le CPU, pleine bande passante (~64 Go/s bidirectionnel)
- **Slot 2 (PCIEX16_2) :** PCIe 5.0 depuis le CPU, partage le même pool de 16 voies. Quand les deux slots sont occupés, ils se bifurquent automatiquement en **x8/x8**
- **Slot 3 :** PCIe 4.0 x4 depuis le chipset, utilisable mais limité (~8 Go/s)

**Point clé :** les slots 1 et 2 partagent les 16 voies GPU du CPU. On obtient soit 1 GPU en x16, soit 2 GPU en x8/x8. Le BIOS supporte aussi des modes de bifurcation incluant x8/x4/x4 et même x4/x4/x4/x4 sur un seul slot (pour des splitters de riser spécialisés).

### Comment Loger Réellement 4 GPU

Pour une config à 4 GPU sur cette carte, l'allocation de voies réaliste est :

**Option A, Meilleur équilibre (recommandée) :**
- Slot 1 + Slot 2 : x8/x8 Gen5 → **GPU 1 + GPU 2** (via risers, les cartes étant trop épaisses pour tenir côte à côte)
- Slot 3 (chipset) : x4 Gen4 → **GPU 3**
- **GPU 4 :** nécessite un riser de bifurcation sur le slot 1 ou 2 (en divisant le x8 en x4/x4), ou l'usage d'un switch PCIe. Certains builders utilisent alternativement le slot M.2 avec un adaptateur M.2-vers-PCIe x4 pour la 4e carte.

**Option B, Densité maximale via bifurcation :**
- Slot 1 : bifurcation x4/x4/x4/x4 → **4 GPU en x4 Gen5** chacun (~16 Go/s bidirectionnel)
- Nécessite une carte riser de bifurcation quadruple et un support BIOS adapté

**Option C, Compromis pratique (le plus courant dans les vrais builds) :**
- Slot 1 : x8 Gen5 → **GPU 1**
- Slot 2 : x8 Gen5 → **GPU 2**
- Slot 3 (chipset) : x4 Gen4 → **GPU 3**
- **GPU 4 :** sur un riser via une bifurcation du slot 1 ou du slot 2 (x4 Gen5), en acceptant qu'un GPU passe en x4

Toutes ces options fonctionnent pour l'inférence LLM. Voici pourquoi :

### Pourquoi Ça Compte à Peine pour l'Inférence LLM

La bande passante PCIe est en grande partie hors sujet pour les workloads d'inférence. Voici pourquoi :

1. **Le chargement du modèle est une opération ponctuelle.** On charge 70 Go de poids en VRAM une seule fois. Même en x4 Gen4 (~8 Go/s), charger 32 Go de poids prend ~4 secondes au lieu de ~0,5 seconde. On ne fait ça qu'une fois.

2. **La génération de tokens est limitée par le calcul GPU**, pas par le transfert PCIe. Le goulot d'étranglement, c'est la multiplication matricielle à l'intérieur du GPU, pas le déplacement de données sur le bus.

3. **Le parallélisme tensoriel multi-GPU** (vLLM, llama.cpp) communique bien entre les GPU pendant l'inférence, mais le volume est faible comparé à l'entraînement. Sur AM5 sans NVLink, la communication inter-GPU passe par le CPU/PCIe, plus lent que le NVLink, mais suffisant pour de l'inférence à des dizaines de tokens/seconde.

4. **Les benchmarks montrent une différence de 0-4%** entre Gen5 x16 et Gen4 x4 pour le débit d'inférence LLM en mono-GPU. En multi-GPU, l'écart peut monter à 5-15% selon la taille du modèle et la stratégie de parallélisme, mais c'est rarement le goulot d'étranglement.

**L'Alternative Threadripper :** si les voies PCIe deviennent réellement un goulot d'étranglement (par ex. des workloads d'entraînement lourds avec de grosses synchronisations de gradients), la plateforme adaptée est l'AMD Threadripper PRO 7000 avec WRX90 (128 voies PCIe 5.0, 32 par GPU pour 4 cartes, DDR5 8 canaux jusqu'à 2 To). Des builds quad-5090 confirmés existent sur Threadripper avec x16 plein par GPU et refroidissement liquide sur-mesure. Mais le CPU seul coûte ~3 000 €+ et les cartes mères ~1 000 €+. Pour des builds centrés inférence, l'AM5 est le choix pragmatique, on sacrifie la largeur de voie mais on garde 95%+ du débit d'inférence pour une fraction du coût de plateforme.

---

## Est-ce que les 4 GPU Tourneront à la Même Vitesse ?

**Réponse courte : non.** C'est la question la plus fréquente sur le multi-GPU en AM5, et elle mérite une réponse franche.

### L'Asymétrie

Sur la ProArt X870E, les 4 GPU reçoivent une bande passante PCIe inégale :

| GPU | Slot | Lien PCIe | Bande passante | Vitesse relative |
|-----|------|-----------|-----------|----------------|
| GPU 1 | Slot 1 CPU | x8 Gen5 | ~32 Go/s | Rapide |
| GPU 2 | Slot 2 CPU | x8 Gen5 | ~32 Go/s | Rapide |
| GPU 3 | Slot chipset | x4 Gen4 | ~8 Go/s | Plus lent |
| GPU 4 | Bifurcation/adaptateur | x4 Gen5 ou Gen4 | ~8-16 Go/s | Plus lent |

Les GPU eux-mêmes (cœurs CUDA, VRAM GDDR7, fréquences) sont identiques. C'est le **tuyau entre le GPU et le CPU** qui varie.

### Quand Est-ce que Ça Compte ?

**Inférence mono-modèle (Ollama, un modèle sur un GPU) :** pas du tout. Le modèle se charge en VRAM une fois, ensuite c'est du calcul purement interne au GPU. Un GPU en x4 Gen4 génère des tokens exactement à la même vitesse qu'un GPU en x8 Gen5.

**Parallélisme tensoriel multi-GPU (vLLM, llama.cpp avec 4 GPU sur un modèle) :** ça compte. Les GPU échangent des données d'activation à chaque token. Le lien le plus lent (x4 Gen4 à ~8 Go/s) devient le goulot d'étranglement car les GPU plus rapides l'attendent. Impact estimé : **débit inférieur de 5-15%** par rapport à 4 GPU à bande passante égale.

**Service multi-modèles (des modèles différents sur des GPU différents) :** ça ne compte pas. Chaque GPU travaille indépendamment. Mettez votre modèle le plus utilisé sur un GPU à voie rapide si vous voulez une latence optimale en mono-modèle.

### Trois Façons d'Égaliser

**Option 1 : Threadripper (la vraie réponse), +3 500-4 000 €**

L'AMD Threadripper PRO 7975WX + carte mère WRX90 donne 128 voies PCIe 5.0. Chaque GPU obtient un slot x16 complet. Pas de bifurcation, pas de compromis, pas d'asymétrie. C'est ce qu'utilisent les stations de travail IA commerciales (BIZON, Lambda, Puget Systems). Si vous dépensez 8 800 € en GPU, les 3 500 € supplémentaires pour le Threadripper valent sans doute le coup pour un système correctement équilibré.

| Composant | Modèle | Prix estimé |
|-----------|-------|-------------|
| CPU | AMD Threadripper PRO 7975WX (32C/64T) | ~3 500 € |
| Carte mère | ASUS WRX90E SAGE ou Gigabyte WRX90 AERO D | ~1 000-1 200 € |
| RAM | Vos kits DDR5 fonctionnent (4 des 8 canaux occupés) | 0 € (réutilisation) |

**Option 2 : Forcer tous les slots en Gen4 x4 (gratuit, tout égal)**

Dans le BIOS, forcer chaque slot PCIe en Gen4. Les 4 GPU tournent alors tous à la même ~8 Go/s. On perd de la bande passante absolue mais on gagne en **symétrie**, aucun GPU n'attend un voisin plus lent pendant le parallélisme tensoriel. Pour l'inférence, le tuyau à ~8 Go/s est rarement le goulot d'étranglement de toute façon.

C'est la stratégie d'égalisation la plus simple et la moins chère. Contre-intuitif mais efficace.

**Option 3 : Compensation logicielle avec tensor-split (gratuit, malin)**

llama.cpp et vLLM supportent tous deux la répartition pondérée des tenseurs. On donne plus de couches de modèle aux GPU rapides et moins aux GPU lents :

```bash
# llama.cpp : répartition du poids sur 4 GPU
# Les GPU 1-2 (x8 Gen5) reçoivent plus de couches, les GPU 3-4 (x4) moins
./llama-server \
  -m models/llama-3.1-70b-Q5_K_M.gguf \
  --n-gpu-layers 99 \
  --tensor-split 30,30,20,20

# vLLM : utilise le parallélisme tensoriel automatiquement
# Contrôle moins fin, mais gère raisonnablement bien l'asymétrie
vllm serve <modèle-70B-actuel-du-leaderboard> \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.90
```

Le GPU lent reçoit moins de travail, termine en même temps que les rapides, et personne n'attend. C'est l'approche la plus pragmatique sur AM5.

### Tableau Comparatif

| Approche | Coût supplémentaire | Les 4 égaux ? | Débit global |
|----------|-----------|--------------|-------------------|
| Threadripper WRX90 | +3 500-4 000 € | Oui (4×x16 Gen5) | 100% |
| AM5 + forcer Gen4 x4 | 0 € | Oui (4×x4 Gen4) | ~92-95% |
| AM5 + réglage tensor-split | 0 € | Compensé | ~90-95% |
| AM5 + ne rien faire | 0 € | Non | ~85-95% |

### Ma Recommandation

Pour l'**inférence** (mon usage principal) : rester sur AM5, utiliser `--tensor-split` pour compenser. La différence de 5-10% face au Threadripper ne justifie pas 3 500 €+.

Pour l'**entraînement** (le LoRA/QLoRA passe, pas le fine-tuning complet) : si vous avez un jour besoin d'entraînement multi-GPU sérieux avec de grosses synchronisations de gradients, vendez la combo AM5 et passez au Threadripper. L'asymétrie PCIe fait beaucoup plus mal à l'entraînement qu'à l'inférence.

---

## Risers PCIe 5.0 : Toute l'Histoire

### Pourquoi les Risers PCIe 5.0 ont Mauvaise Réputation

Le PCIe 5.0 tourne à **32 GT/s** (giga-transferts par seconde), le double des 16 GT/s du Gen4. À ces fréquences :

- **L'intégrité du signal se dégrade plus vite** avec la longueur du câble, les courbures, et un blindage médiocre
- **Les risers bon marché introduisent du bruit** qui cause des échecs de négociation de lien, des crashs sous charge, ou une dégradation automatique vers Gen4/Gen3
- Le **design multi-PCB de la RTX 5090** (surtout Founders Edition) agit déjà comme une sorte de riser interne, ce qui aggrave les difficultés d'intégrité du signal

Des testeurs comme der8auer, Igor's Lab et Hardware Canucks ont documenté des instabilités avec des risers bas de gamme sur les cartes RTX série 50, en particulier quand le BIOS est réglé sur génération PCIe « Auto ».

### Mais Ce N'est Pas une Fatalité

Avec des **risers PCIe 5.0 de qualité**, ça marche bien. Les facteurs clés :

| Facteur | Bon | Mauvais |
|--------|------|-----|
| Longueur du câble | 15-30 cm | >40 cm |
| Blindage | Multi-couche, couverture complète | Fin ou partiel |
| Connecteurs | Plaqués or, renforcés | Lâches ou bon marché |
| Courbure | Courbes douces uniquement | Plis serrés, écrasements |
| Marque | LINKUP, Thermaltake, ADT-Link | Sans marque, AliExpress |

### Ma Recommandation

**Choix principal :** LINKUP AVA5 PCIe 5.0 (versions 20-30 cm). En 2026, ce sont les plus largement validés pour les builds RTX 5090. Ils maintiennent la pleine bande passante (128 Go/s aux tests 3DMark) même avec des courbures modérées. Prix : ~55-80 € pièce.

**Alternative économique :** des risers PCIe 4.0 de qualité + forcer le Gen4 dans le BIOS. Perte de performance : 1-3% en inférence. Gain de stabilité : significatif. Prix : ~25-40 € pièce. De nombreux builds multi-5090 tournent ainsi sans problème.

**Stratégie de repli :** commencer avec des risers Gen5, régler le BIOS sur Auto. En cas d'instabilités (GPU non détecté, crashs sous charge soutenue, GPU-Z montrant un lien Gen3/4 inattendu), forcer le Gen4 dans le BIOS. On ne perd presque rien pour les workloads LLM.

### Protocole de Validation

Après montage, vérifier avec :

```bash
# Vérifier la vitesse et la largeur du lien PCIe
nvidia-smi -q | grep -A 5 "PCI"

# Test de charge : inférence soutenue pendant 24h
# (utiliser votre workload réel, pas un benchmark synthétique)
ollama run <votre-modèle-70B>:q4_K_M

# Surveiller les températures sous charge
watch -n 1 nvidia-smi
```

Vérifier que GPU-Z montre bien la génération attendue (Gen5 x8 ou Gen4 x16/x8) et qu'elle ne fluctue pas sous charge.

---

## Alimentation Électrique : Stratégie Multi-Alim

### Budget de Puissance

| Composant | Consommation de pointe |
|-----------|-----------|
| RTX 5090 Vanguard SOC (×4) | 4 × 575W = 2 300W |
| Ryzen 9 9950X (PBO) | ~200W |
| RAM, SSD, ventilateurs, carte mère | ~80W |
| **Total pic** | **~2 580W** |
| **Avec pics transitoires** | **~2 800-3 000W** |

Une seule alim ATX ne peut pas gérer ça. Il faut un montage multi-alim.

### Ma Configuration d'Alims

```
Alim 1 : HP 1200W Server (80+ Platinum)
  → Carte mère 24 broches (via breakout ATX)
  → CPU 8 broches
  → GPU 1 (16 broches natif)
  → GPU 2 (16 broches natif)

Alim 2 : HP 1200W Server (80+ Platinum)
  → GPU 3 (16 broches natif)
  → GPU 4 (16 broches natif)

Alim 3 : Corsair SF1000 (secours / marge)
  → Disponible si équilibrage de charge nécessaire
  → Ou utilisée pour les ventilateurs + périphériques
```

### Notes sur les Alims Serveur HP

Les unités HP ProLiant 1200W (DPS-1200FB ou HSTNS-PL11) sont **excellentes** pour les builds multi-GPU :

- **Rendement 80+ Platinum**, conçues pour un fonctionnement datacenter 24/7
- **Rail 12V unique à 100A**, sortie propre, haut courant
- **Très bon marché** sur le marché de l'occasion (~30-50 € pièce)
- Largement utilisées dans les builds de mining et d'IA depuis 2017

**Ce qu'il faut pour les faire fonctionner :**

- **Cartes breakout** (~15-30 € sur AliExpress/Amazon) qui convertissent le connecteur d'alim serveur en câbles ATX/PCIe standards
- **Câbles 16 broches (12V-2×6) de qualité** pour la RTX 5090. Ne PAS utiliser d'adaptateurs Molex-vers-16-broches bon marché, ils peuvent fondre sous une charge soutenue de 575W. Prendre des câbles calibrés pour le courant.
- Un moyen de **synchroniser le démarrage** entre les alims (certaines cartes breakout le gèrent, ou utiliser un fil de pontage sur le fil vert de l'ATX 24 broches)

### L'Alim Pieuvre (AliExpress 1600W)

Ces alims « pieuvre » sont essentiellement des cartes breakout d'alim serveur avec câbles intégrés, souvent vendues pour le mining. Elles fonctionnent, mais :

- **La qualité varie énormément**, tester la stabilité de la tension sous charge avant de lui faire confiance avec des 5090
- À utiliser en **complément**, pas en alimentation principale
- Bonnes pour alimenter 1-2 GPU pendant que les unités HP gèrent le reste

### Infrastructure Électrique

**Critique :** 4x5090 à pleine charge tire ~2,5-3 kW du secteur. Les circuits européens standards 16A / 230V fournissent 3,68 kW max, on est proche de la limite.

Recommandations :
- **Circuit dédié 20A ou 32A** pour le rig
- Ne jamais partager le circuit avec des radiateurs d'appoint, bouilloires, ou autres appareils très gourmands
- Un **onduleur (UPS) est optionnel mais malin**, pas pour l'autonomie, mais pour une protection d'arrêt propre
- Envisager de **sous-volter les GPU** (voir plus bas) pour réduire la consommation de 20-30% avec une perte de performance minime

---

## Refroidissement : Open-Air vs. Boîtier Fermé

### Pourquoi l'Open-Air l'Emporte pour ce Build

Le MSI Vanguard SOC est une carte **quad-slot** (~76 mm d'épaisseur, 336 mm de long). Quatre de ces cartes dans un boîtier fermé, même une tour complète, créent un cauchemar thermique :

- Les cartes se toucheraient ou se chevaucheraient physiquement dans des boîtiers ATX standards
- Aucune circulation d'air entre les GPU
- La température ambiante du boîtier monte à 50-60°C sous charge soutenue
- Le throttling thermique du GPU s'enclenche, réduisant la vitesse d'inférence

**Avantages de l'open-air :**

- Espacer chaque GPU de 2-3 largeurs de slot
- Accès direct à l'air ambiant sur toutes les faces
- Facile d'ajouter des ventilateurs directionnels entre les cartes
- Maintenance et changement de cartes triviaux
- Températures GPU 10-15°C plus fraîches qu'en boîtier fermé (mesuré sur des builds similaires)

### L'Approche Mining de Crypto (Boîtier Fermé + Ventilateurs Industriels)

Certains mineurs utilisent des boîtiers fermés avec des **souffleurs industriels à haut débit** (200-300 mm, 100+ CFM) créant une pression positive massive. Ça fonctionne pour :

- Les environnements poussiéreux (entrepôts, garages)
- L'évacuation forcée de l'air chaud par des conduits
- Le fonctionnement 24/7/365 sans surveillance

Pour un **rig d'inférence LLM à domicile/en studio**, l'open-air est plus simple, moins cher, et plus silencieux. L'approche du souffleur industriel ajoute de la complexité et du bruit pour un bénéfice marginal dans un environnement intérieur propre.

### Ma Configuration de Refroidissement

```
[VENTILATEURS D'ENTRÉE]  →  [GPU 1]  →  [VENTILATEUR]  →  [GPU 2]  →  [VENTILATEUR]  →  [GPU 3]  →  [VENTILATEUR]  →  [GPU 4]  →  [ÉCHAPPEMENT]

Bas du châssis : carte mère + CPU (Noctua NH-D15)
Entre chaque GPU : 1-2 ventilateurs 120mm (Arctic P12 ou Noctua NF-A12x25)
Côté/dessus : ventilateurs 140mm additionnels pour la circulation générale
```

**Astuce :** monter les GPU à l'horizontale (à plat) si le châssis le permet. Ça évite l'affaissement des GPU et garde les radiateurs optimaux (la chaleur monte naturellement).

### Sous-Voltage : de la Performance Gratuite

La RTX 5090 Vanguard SOC sort d'usine avec des courbes d'overclock agressives. Le sous-voltage réduit significativement la consommation et la chaleur :

```bash
# Exemple : plafonner la puissance à 80% (économise ~115W par carte)
nvidia-smi -i 0 -pl 460  # GPU 0
nvidia-smi -i 1 -pl 460  # GPU 1
nvidia-smi -i 2 -pl 460  # GPU 2
nvidia-smi -i 3 -pl 460  # GPU 3
```

Résultats attendus à une limite de puissance de 80% :
- Consommation : ~460W vs. 575W par carte (économise 460W au total sur 4 GPU)
- Perte de performance : 3-8% en inférence (à peine perceptible en tokens/s)
- Baisse de température : 5-10°C par carte
- La consommation totale du rig passe de ~2 600W à ~2 100W, bien plus amical pour votre circuit

Pour des sessions d'inférence longues, c'est la **meilleure optimisation possible**.

---

## Configuration BIOS

### Réglages Essentiels

| Réglage | Valeur | Pourquoi |
|---------|-------|-----|
| Génération PCIe | Gen 4 (ou Auto → repli Gen 4) | La stabilité prime sur la bande passante |
| XMP / EXPO | Activé (DDR5-6000) | Utiliser la vitesse nominale de votre RAM |
| Resizable BAR | Activé | Permet au GPU d'accéder au mapping complet de la VRAM |
| Above 4G Decoding | Activé | Requis pour le multi-GPU + grande VRAM |
| SR-IOV | Désactivé (sauf usage de vGPU) | Simplicité |
| CSM | Désactivé | Boot UEFI, OS moderne |
| Bifurcation PCIe | x8x8 sur le slot 1 (si disponible) | Permet 2 GPU sur les voies CPU |
| C-States | Activé | Économie d'énergie au repos |
| PBO | Activé, mode éco optionnel | Équilibre perf CPU vs. chaleur |

### Configuration RAM

Avec 4×48 Go (192 Go au total), les quatre slots DIMM sont occupés :

- Slots **A2 + B2** en premier (dual-channel primaire), votre premier kit
- Slots **A1 + B1** en second, votre second kit
- Le profil XMP/EXPO devrait détecter automatiquement le DDR5-6000 CL30
- En cas d'instabilité avec 4 barrettes à 6000 MHz, redescendre à 5600 MHz, les configs à 4 DIMM sont plus difficiles à faire tourner à haute fréquence sur AM5

**Pourquoi 192 Go de RAM comptent :** pour l'inférence LLM, la RAM système est utilisée pour :
- Le buffer de chargement du modèle avant le transfert vers le GPU
- Le débordement du cache KV (quand la VRAM est pleine)
- L'offload CPU des couches de modèle (offload partiel `--n-gpu-layers` de llama.cpp)
- Faire tourner ComfyUI, le serveur vLLM, le monitoring, et l'OS simultanément

192 Go donne une marge confortable pour tout ça. 96 Go fonctionnerait mais devient serré avec plusieurs gros modèles chargés.

---

## Stack Logicielle

### Inférence Cœur

```bash
# Ollama (le plus rapide à démarrer, détecte automatiquement les 4 GPU)
ollama serve
ollama run qwen3:32b          # tient dans 1 GPU
ollama run qwen3:72b-q4_K_M   # s'étale automatiquement sur 2-3 GPU

# vLLM (le meilleur pour les requêtes concurrentes + le parallélisme tensoriel)
vllm serve <modèle-Tier2-actuel-du-leaderboard> \
  --tensor-parallel-size 4 \
  --gpu-memory-utilization 0.90

# llama.cpp (la meilleure flexibilité, format GGUF, offload CPU partiel)
./llama-server \
  -m models/qwen3-70b-Q5_K_M.gguf \
  --n-gpu-layers 99 \
  --tensor-split 30,30,20,20  # compense l'asymétrie de voies AM5
```

### LiteLLM : Proxy API Unifié (essentiel pour les frameworks d'agents)

LiteLLM expose tous vos modèles locaux derrière un seul endpoint compatible OpenAI.
Tout framework d'agent (Hermes Agent, Claude Code, LangChain, CrewAI) parle à une seule URL.

```bash
pip install litellm
litellm --model ollama/qwen3:32b --port 4000
# ou pointer vers vLLM :
litellm --model openai/qwen3-70b --api-base http://localhost:8000/v1 --port 4000
# maintenant n'importe quelle app tape sur http://localhost:4000 peu importe le backend
```

### Hermes Agent : Couche Agentique

[Hermes Agent](https://github.com/NousResearch/hermes-agent) (NousResearch, sorti en février 2026) est un framework d'agent autonome, pas juste un modèle. Il tourne au-dessus de n'importe quel backend LLM local et gère les appels d'outils, la mémoire, et le raisonnement multi-étapes :

```bash
curl -fsSL https://raw.githubusercontent.com/NousResearch/hermes-agent/main/scripts/install.sh | bash
# le pointer vers votre vLLM/Ollama local via LiteLLM :
hermes config set provider openai
hermes config set base_url http://localhost:4000
hermes config set model qwen3:32b
hermes run "analyse toutes les images dans ~/shots/ et crée une shot list"
```

Fonctionnalités clés : 40+ outils intégrés, sous-agents pour des chantiers parallèles, mémoire de session, timeout socket de 30 min pour l'inférence locale à grand contexte. Les modèles Hermes fine-tunés en dessous sont entraînés pour l'appel de fonctions structuré (format `<tool_call>` / `<tool_response>`), plus fiable que les modèles d'instruction de base pour les tâches agentiques.

### Que Peut Faire Tourner 128 Go de VRAM ?

> **Les modèles changent plus vite que le hardware. Ne pas utiliser cette section comme une liste de courses.**
> Vérifier plutôt ceci, mis à jour en temps réel :
> - [openrouter.ai/rankings](https://openrouter.ai/rankings), pondéré par l'usage, ce que les gens font réellement tourner
> - [lmarena.ai](https://lmarena.ai), ELO de préférence humaine
> - [artificialanalysis.ai](https://artificialanalysis.ai), agrégateur de benchmarks

### Ce que Débloquent 128 Go de VRAM, par Niveau

Le hardware définit quels niveaux sont atteignables. Les modèles précis à chaque niveau changent constamment.

| Niveau | VRAM nécessaire | Ce qui tient | Vitesse typique | Exemples (avril 2026, vérifier à jour) |
|------|------------|-----------|--------------|----------------------------------------|
| **Tier 0, Agents rapides** | 4-18 Go (1 GPU) | 7B-32B dense ou MoE | 50-200+ tok/s | Qwen3 8B, Gemma 4 9B, DeepSeek-R1 distill 7B |
| **Tier 1, Agents de qualité** | 18-40 Go (1-2 GPU) | 30B-32B dense | 30-80 tok/s | Qwen3 32B, variantes Qwen3.6 Plus, distills MiniMax |
| **Tier 2, Équivalent frontier** | 40-80 Go (2-3 GPU) | 70B dense, 200B+ MoE | 15-50 tok/s | Meilleurs modèles open-weight du moment (vérifier le leaderboard) |
| **Tier 3, Capacité max** | 80-128 Go (3-4 GPU) | 200B+ MoE en meilleure quant | 10-30 tok/s | Kimi K2.5, GLM-5.1, MiniMax M2.7 en Q3-Q4 |
| **Débordement (CPU)** | 128 Go VRAM + 192 Go DDR5 | 400B-670B offload partiel | 5-15 tok/s | DeepSeek-V3 complet, futurs modèles 400B+ |
| **Vidéo, 1 GPU dédié** | 32-40 Go | LTX Video 2.3, WanVideo 2.2 14B |, | Fixé par l'architecture du modèle, moins volatil |

**Règle générale :** le leaderboard des modèles tourne toutes les 4-8 semaines. La structure des niveaux ne change pas. Une carte Tier 2 aujourd'hui fait tourner quel que soit le modèle Tier 2 dans 18 mois.

**Pourquoi 192 Go de DDR5 comptent pour les gros modèles :** quand un modèle MoE 200B+ ou 400B+ déborde au-delà des 128 Go de VRAM, les couches en trop tournent sur la RAM CPU. Avec 192 Go de DDR5-6000, le repli est assez rapide (~8-15 tok/s) pour être utilisable. Sur une machine avec 32-64 Go de RAM, le même débordement tombe à 2-3 tok/s.

Le point idéal, ce sont les modèles **70B Q5-Q8**, ils tiennent entièrement en VRAM sur 2-4 GPU avec une inférence rapide.

### Génération Image/Vidéo (ComfyUI)

**ComfyUI v0.16.1+**, VRAM dynamique activée par défaut. Mettre à jour avant tout le reste.

**Modèles actuels (avril 2026) :**

| Modèle | Type | VRAM (FP8/quant) | Notes |
|-------|------|-----------------|-------|
| Flux.2 Dev/Schnell | Image | ~12-16 Go | NVFP4 = 3× plus rapide que FP16 sur 5090 |
| Flux.2 Pro | Image | ~8-12 Go | Variante Flux plus rapide, step-distilled |
| WanVideo 2.2 (14B) | Vidéo | ~40 Go FP8, ~20 Go (720p) | Meilleure qualité de mouvement vidéo open-source |
| WanVideo 2.2 (5B) | Vidéo | ~8-16 Go | Plus rapide, plus léger, toujours bon |
| LTX Video 2.3 (22B) | Vidéo + Audio | ~32 Go+ (officiel), ~12-24 Go FP8 | Génère audio + vidéo simultanément |

```bash
# Layout multi-workflow 4x5090 :
#   GPU 0 : WanVideo 2.2 14B (FP8, ~40 Go), carte vidéo dédiée
#   GPU 1-2 : inférence modèle Tier 2 (70B tensor parallel, ~55 Go)
#   GPU 3 : batches d'images Flux.2 + entraînement SD 3.5 / LoRA
#
# ComfyUI sélectionne le GPU via CUDA_VISIBLE_DEVICES :
CUDA_VISIBLE_DEVICES=0 python -m main --port 8188  # nœud vidéo
CUDA_VISIBLE_DEVICES=3 python -m main --port 8189  # nœud image
```

**Note LTX Video 2.3 :** le modèle 22B a officiellement besoin de 32 Go+. Une seule RTX 5090 (32 Go) le fait tourner en FP8 avec `--reserve-vram 5`. Avec les [nœuds de gestion VRAM](https://github.com/RandomInternetPreson/ComfyUI_LTX-2_VRAM_Memory_Management), la consommation pic chute d'environ 10×, permettant 800+ frames en 1920×1088 sur une seule carte.

**Note WanVideo 2.2 :** le modèle 14B en FP8 a besoin de ~40 Go, dédier le GPU 0 à ça. La variante 5B tient dans une seule carte (24-32 Go) avec de bons résultats.

### Monitoring

```bash
# Stats GPU en temps réel
watch -n 1 nvidia-smi

# Monitoring détaillé puissance/température
nvidia-smi dmon -s pucvmet -d 5

# Utilisation par GPU pour le service multi-modèles
gpustat --watch
```

---

## Intégration Réseau

Ce rig rejoint mon mesh Tailscale existant :

| Machine | Rôle | IP Tailscale |
|---------|------|-------------|
| MacBook Air M3 | Poste de travail principal | 100.x.x.1 |
| Mac Mini M4 | Serveur (Paris) | 100.x.x.2 |
| **Rig 4x5090** | Calcul GPU | À déterminer |
| PC Windows (existant) | GPU + ComfyUI | 100.x.x.3 |

Ollama ou vLLM expose une API sur le réseau Tailscale, accessible depuis n'importe quelle machine :

```bash
# Depuis le MacBook, interroger le rig 4x5090
curl http://<ip-tailscale-du-rig>:11434/api/generate \
  -d '{"model": "votre-modèle-70b", "prompt": "..."}'
```

---

---

## Portabilité : Emmener le Rig en Déplacement

Un châssis mining open-air n'est pas conçu pour voyager. Mais il peut, avec la bonne approche.

### Vérité sur le Poids

| Composant | Poids approximatif |
|-----------|---------------|
| 4x RTX 5090 Vanguard SOC | ~4× 1,8 kg = ~7,2 kg |
| Châssis open-air (aluminium/acier) | ~3-5 kg |
| Carte mère + CPU + ventirad | ~2 kg |
| 2x alims serveur HP | ~4 kg |
| Alim Corsair SFX | ~0,8 kg |
| RAM + SSD + câbles | ~1 kg |
| **Total** | **~18-20 kg** |

Pas des bagages. Du transport, pas du voyage.

### Option A : Rack en Valise (le Plus Professionnel)

Remplacer le châssis open-air par un **châssis rackmount 4U** (Tupavco TP1846 ou Hydra III), puis le loger dans une **valise de transport rackmount Pelican/SKB** (SKB R914U ou équivalent). C'est l'approche pro audio/vidéo, le même système utilisé pour les serveurs GPU de diffusion en direct.

- Avantages : entièrement enfermé, protégé, empilable, transportable en fret hors gabarit en avion
- Inconvénients : ~400-600 € pour le rack + la valise, aération légèrement moins bonne qu'en open-air
- Les cartes GPU ont quand même besoin d'être rembourrées/fixées individuellement à l'intérieur du châssis 4U

### Option B : Démonter et Transporter Séparément

Pour un transport occasionnel (Paris → LA, résidences) :
1. Retirer les 4 GPU → chacun dans un sac antistatique + manchon mousse → en bagage cabine ou Pelican 1510 rembourré
2. Expédier le châssis + carte mère + alims (non fragile, peut voyager en fret)
3. Remonter sur place (~45 min)

C'est comme voyagent la plupart des cinéastes avec des rigs GPU. Les GPU sont les pièces irremplaçables/coûteuses, elles voyagent avec vous. Le reste est expédié.

### Option C : Accès à Distance (la Meilleure Option pour la Plupart des Voyages)

Laisser le rig à Paris. Y accéder via **Tailscale** depuis n'importe où avec internet :

```bash
# Depuis n'importe où : ssh sur le rig, lancer l'inférence à distance
ssh user@<ip-tailscale-du-rig>
# ou taper directement sur l'endpoint LiteLLM :
curl http://<ip-tailscale-du-rig>:4000/v1/chat/completions ...
```

La plupart des cas d'usage en production ne nécessitent pas de proximité physique. Le rig sert tout le réseau. Pour un tournage à LA, le rig parisien gère les tâches d'agent et l'inférence de modèle pendant la nuit, on se réveille avec les résultats. C'est le vrai avantage de l'intégration Tailscale.

### Compatibilité Électrique (France ↔ USA / International)

| Alim | Tension d'entrée | Usage international |
|-----|---------------|------------------|
| Corsair SF1000 | 100-240V auto | Fonctionne partout |
| HP 1200W Server PSU | 100-240V auto | Fonctionne partout |
| Alim breakout pieuvre | Typiquement 110-240V, **vérifier l'étiquette** | Peut nécessiter un transformateur |

Les alims serveur HP et la Corsair SF sont universelles. Vérifier l'alim pieuvre AliExpress avant de la brancher à l'étranger.

---

## Budget Restant (Ce qu'il Me Reste à Acheter)

Déjà possédés : châssis open-air, alims serveur HP ×2, alim pieuvre, Corsair SF1000, ventilateurs Noctua supplémentaires.

| Élément | Coût estimé | Priorité |
|------|-----------|----------|
| 4x risers PCIe 5.0 (LINKUP AVA5 20-30cm) | 280-350 € | Critique |
| 4-6x ventilateurs Arctic P12 PWM | 50-70 € | Élevée |
| 2x cartes breakout HP + câbles 16 broches | 60-80 € | Élevée |
| Câbles divers, colliers, entretoises | 20-30 € | Moyenne |
| **Total restant** | **410-530 €** | |

---

## Réserves Honnêtes

### Ce que Je Changerais en Recommençant

1. **Threadripper PRO 7000 WRX90** au lieu d'AM5. La situation des voies PCIe sur AM5 est jouable mais pas idéale pour 4 GPU. Le Threadripper donne 128 voies PCIe 5.0 (32 par GPU, donc 4 cartes en x16 complet chacune sans bidouilles de bifurcation), 8 canaux DDR5 (jusqu'à 2 To), et est conçu exactement pour ce cas d'usage. Des builds multi-5090 confirmés existent sur Threadripper avec refroidissement liquide sur-mesure. Différence de coût : ~3 000-4 000 € de plus pour CPU+carte mère. Ça vaut le coup si c'est une machine de production ou si vous prévoyez d'entraîner (pas seulement d'inférer).

2. **RAM ECC** pour les jobs d'inférence longs. La DDR5 grand public peut avoir de rares bit flips pendant des sessions de 24h+. L'ECC les rattrape. Disponible sur Threadripper ; techniquement supporté mais non validé sur la plupart des cartes AM5.

3. **Circuit dédié 32A** dès le départ. Je tourne près de la limite d'un circuit européen standard 16A / 230V (3,68 kW max). Avec le sous-voltage ça va, mais à pleine puissance les 4 GPU peuvent faire sauter le disjoncteur.

4. **GPU au design de référence / blower** plutôt que des cartes à ventirad open-air épais. Le design quad-slot du Vanguard SOC fait de l'espacement physique un défi même sur des châssis open-air. Les cartes au PCB de référence sont plus fines, et les ventirads blower évacuent la chaleur directionnellement plutôt que de rayonner vers les cartes adjacentes. Certains builders à 4 GPU recherchent spécifiquement des cartes de référence compatibles waterblock pour cette raison.

### Risques Connus

- **L'allocation des voies PCIe est le risque n°1.** Les 16 voies GPU d'AM5 réparties sur 2 slots signifient jongler avec la bifurcation, les slots chipset, et éventuellement des adaptateurs PCIe pour loger 4 cartes. Tester chaque GPU individuellement avant la config complète à 4 cartes. Si un GPU échoue à s'initialiser, c'est probablement un problème de voie/slot, pas une carte morte.
- **La stabilité des risers PCIe** est le risque n°2. Risers bon marché + Gen5 + RTX 5090 = potentiel de crashs intermittents frustrants. Dépenser l'argent pour des risers de qualité.
- **Les alims serveur HP sont bruyantes.** Les ventilateurs 40mm d'origine montent à des niveaux de réacteur sous charge. Modification courante : remplacer les ventilateurs d'origine par des Noctua 40mm (NF-A4x20 PWM) pour une réduction de bruit spectaculaire tout en gardant une aération sûre.
- **4 barrettes DDR5-6000 sur AM5** peuvent être capricieuses. L'IMC (contrôleur mémoire intégré) du Zen 5 travaille plus dur avec 4 DIMM occupés. En cas d'erreurs mémoire ou d'échecs de POST, redescendre à 5600 MHz et resserrer les timings.
- **La poussière** dans les montages open-air. Un nettoyage mensuel à l'air comprimé est obligatoire. Garder le rig dans une pièce propre et ventilée, loin des animaux et des tissus.
- **Le partage de slot M.2 avec les voies GPU.** Sur la ProArt X870E, certains slots M.2 partagent la bande passante avec les slots GPU PCIe. Vérifier le manuel pour savoir quel port M.2 utilise les voies CPU vs. les voies chipset. Prioriser le M.2 chipset pour votre second SSD pour éviter de voler de la bande passante GPU.

### Pour Quoi ce Build N'est PAS Fait

- **L'entraînement à grande échelle** (pré-entraînement, fine-tuning complet de modèles 70B+), il faut du NVLink ou de l'InfiniBand pour une synchronisation de gradients efficace. Ce build est centré sur l'inférence et le LoRA/QLoRA.
- **Le silence**, 4x5090 sous charge + alims serveur + ventilateurs supplémentaires = du bruit significatif. C'est une machine utilitaire, pas un PC de salon.
- **Le plug-and-play**, multi-alim, risers, et open-air signifient plus de montage, de dépannage, et de maintenance qu'un PC standard.

---

## Journal de Montage / Ordre d'Assemblage

1. Monter la carte mère sur le châssis open-air
2. Installer le CPU + ventirad NH-D15
3. Installer 4x48 Go de RAM (tous les slots)
4. Installer 2x SSD M.2
5. Connecter les risers aux slots PCIe (tester un seul GPU d'abord avant les quatre)
6. Monter le GPU 1 directement ou sur un riser court, démarrer, installer OS + drivers
7. Ajouter les GPU 2-4 un à la fois, en testant la stabilité après chacun
8. Connecter les alims (unités HP via cartes breakout)
9. BIOS : activer Above 4G, Resizable BAR, régler la génération PCIe, XMP
10. Installer Ubuntu Server 24.04 ou similaire (headless, SSH)
11. Installer les drivers NVIDIA + toolkit CUDA
12. Installer Ollama / vLLM / llama.cpp
13. Lancer un test de charge de 24h avec le modèle cible
14. Sous-volter, vérifier les températures, réglage final

**Règle d'or :** ajouter un GPU à la fois. Si quelque chose est instable avec 4 GPU, retirer des GPU jusqu'à la stabilité, puis rajouter. Isoler le problème avant de déboguer.

---

## Performance Attendue

Basée sur des builds 4x5090 comparables :

| Workload | Performance attendue |
|----------|---------------------|
| 70B Q4_K_M (Tier 2, mono-GPU, tenue partielle, un peu de débordement KV) | ~25-35 tok/s |
| 70B Q4_K_M (Tier 2, tensor parallel 2 GPU) | ~50-70 tok/s |
| 70B Q4_K_M (Tier 2, tensor parallel 4 GPU) | ~80-120 tok/s |
| 70B Q8 (Tier 2, 4 GPU, sans perte de quant) | ~50-70 tok/s |
| 7-14B Q4 (Tier 3, mono-GPU) | 150-250+ tok/s |
| Génération d'image Flux.2 (NVFP4 sur 5090) | ~1-3 sec/image |
| WanVideo 2.2 14B FP8 (GPU dédié) | varie selon la résolution |
| LTX Video 2.3 22B FP8 (mono-GPU + gestion VRAM) | varie selon la longueur |
| Fine-tuning LoRA (QLoRA, 70B, 4 GPU) | ~4-6× vs. mono-GPU |

Ce sont des estimations approximatives. Les chiffres réels dépendent de la quantification, de la taille de batch, de la longueur de prompt, du cache KV, et de la version logicielle. Pour des benchmarks à jour par modèle, vérifier [artificialanalysis.ai](https://artificialanalysis.ai).

---

---

## Toutes les Alternatives Comparées

> TL;DR : le 4x RTX 5090 sur AM5 est la seule config prosumer qui combine 128 Go de VRAM CUDA, l'écosystème CUDA complet (ComfyUI/Flux.2/LTX-2/vLLM/PyTorch), et un seuil de rentabilité face au cloud atteint en moins de 18 mois. Chaque alternative échoue sur au moins un de ces critères.

### La Matrice Complète de Benchmarks

| Config | VRAM | BW/carte | 70B Q4 tient ? | CUDA | ComfyUI Flux.2 | LTX Video 2 | Coût GPU estimé |
|--------|------|---------|-------------|------|----------------|-------------|---------------|
| RTX 3090 × 1 | 24 Go | 936 Go/s | Non | Oui | Oui | Partiel (offload) | ~500 € occasion |
| RTX 3090 × 4 | 96 Go | 936 Go/s | Oui | Oui | Oui | Oui (1 carte) | ~2 000 € occasion |
| RTX 4090 × 1 | 24 Go | 1 008 Go/s | Non | Oui | Oui | Partiel (offload) | ~1 500 € |
| RTX 4090 × 2 | 48 Go | 1 008 Go/s | Juste | Oui | Oui | Oui (1 carte) | ~3 000 € |
| RTX 4090 × 4 | 96 Go | 1 008 Go/s | Oui | Oui | Oui | Oui | ~6 000 € |
| RTX 5090 × 1 | 32 Go | 1 792 Go/s | Partiel (Q3 seulement) | Oui | Oui | Oui (juste) | ~4 100 € |
| RTX 5090 × 2 | 64 Go | 1 792 Go/s | Oui (confortable) | Oui | Oui | Oui | ~8 200 € |
| **RTX 5090 × 4 AM5 (ce build)** | **128 Go** | **1 792 Go/s** | **Oui + marge** | **Oui** | **Oui** | **Oui (GPU dédié)** | **~8 800 €** |
| RTX 5090 × 4 Threadripper | 128 Go | 1 792 Go/s | Oui + marge | Oui | Oui | Oui | ~8 800 € GPU + 4 500 € plateforme |
| Mac Mini M4 Pro (64 Go) | 64 Go unifié | 273 Go/s | Partiel (MLX seulement) | Non | Non (Metal seulement) | Non (Metal seulement) | ~1 800 € |
| Mac Studio M4 Max (128 Go) | 128 Go unifié | 410 Go/s | Oui (MLX seulement) | Non | Non | Non | ~3 800 € |
| Mac Studio M4 Ultra (192 Go) | 192 Go unifié | 819 Go/s | Oui + 405B aussi | Non | Non | Non | ~5 999 € |
| Mac Mini M4 + eGPU 5090 |, |, | Non (voir plus bas) | Non | Non | Non | ~4 000 €+ gaspillés |

**Tokens/s sur Qwen3 70B Q4_K_M (multi-GPU, tensor parallel) :**

| Config | Tok/s approx. | Notes |
|--------|---------------|-------|
| 4x RTX 3090 | ~35-45 | La bande passante PCIe 3.0 limite la synchro inter-GPU |
| 2x RTX 4090 | ~35-50 | 48 Go tient, marge de cache KV minime |
| 4x RTX 4090 | ~55-75 | 96 Go, confortable ; bande passante vieillissante |
| 2x RTX 5090 | ~55-75 | 64 Go, confortable |
| **4x RTX 5090 (ce build)** | **~80-120** | **128 Go, cache KV complet, tensor split optimal** |
| Mac Studio M4 Ultra | ~30-40 | MLX seulement, pas d'écosystème CUDA |

### Pourquoi Pas Chaque Alternative

**4x RTX 3090 (l'option multi-GPU la moins chère, ~2 000 € au total)**

Sur le papier, ça couvre l'inférence 70B. En pratique :

- **Le PCIe 3.0** partout, la communication inter-GPU est deux fois plus lente qu'en Gen4, et 4× plus lente qu'en Gen5. Chaque synchro de token en tensor-parallel paie cette pénalité
- **936 Go/s de bande passante par carte** contre 1 792 Go/s sur la 5090. L'inférence est de la bande passante mémoire pure. On paie la moitié de la vitesse
- **LTX Video 2** tourne à peine avec un offload mémoire agressif. L'encodeur de texte Gemma 3 12B a besoin de ~24-27 Go à lui seul ; sur une carte de 24 Go, on offload l'encodeur vers la RAM à chaque génération
- **Retard des fonctionnalités CUDA**, NVIDIA dépriorise Ampere (série 30) pour les nouveaux cuDNN, Flash Attention 3, et kernels NVFP4. Les optimisations vLLM et ComfyUI ciblent d'abord Ada (série 40) et Blackwell (série 50)
- **Même consommation pour moitié moins de performance**, une 3090 tire ~350W ; 4× = 1 400W pour ~35-45 tok/s contre les 80-120 tok/s du rig 5090

**Coût électrique (France, ~0,22 €/kWh, usage 8h/jour) :**

| Config | Consommation de pointe | 8h/jour mensuel | Coût/mois |
|--------|-----------|----------------|------------|
| 4x RTX 3090 | ~1 400W + système | ~370 kWh | ~81 € |
| 4x RTX 5090 plein | ~2 580W + système | ~620 kWh | ~136 € |
| 4x RTX 5090 sous-volté (460W) | ~1 960W | ~470 kWh | ~103 € |
| DGX Spark | ~200W | ~48 kWh | ~11 € |
| Mac Studio M4 Ultra | ~75W | ~18 kWh | ~4 € |

Le rig de 3090 tire presque autant que le rig de 5090 pour environ moitié moins de vitesse d'inférence. Le hardware « pas cher » ne coûte pas si peu à faire tourner.

*Verdict : très bien si votre budget total est de 5 000 €. Mauvais choix pour un rig prosumer dédié construit pour durer 4 ans.*

**2x ou 4x RTX 4090**

La recommandation la plus courante en 2024. Mi-2026, le calcul a changé :

- **Le plafond de 24 Go est réel**, Qwen3 70B Q4_K_M a besoin de ~40 Go. Sur 2x 4090 (48 Go), le modèle tient mais il ne reste presque pas de budget de cache KV. Les contextes longs commencent à déborder vers la RAM, les vitesses chutent à ~8-15 tok/s
- **LTX Video 2** tient sur une seule 4090 avec des nœuds de gestion mémoire, mais tout juste. Tout workflow simultané Flux.2 + LTX-2 épuise la VRAM sur un système à 2 cartes
- **4x 4090 (96 Go)** est plus convaincant, mais à ~6 000 € de coût GPU contre ~8 800 € pour 4x 5090, les 32% de dépense supplémentaire achètent 78% de bande passante en plus par carte et 33% de VRAM en plus. La 5090 est un meilleur rapport qualité-prix au palier 4 cartes
- **Prix de rue**, la 4090 n'a pas significativement baissé malgré le lancement de la 5090. On paie des prix proches de la 5090 pour de la bande passante génération Ada

*Verdict : optimal pour des configs 1-2 cartes. Dépassé au palier 4 cartes par la 5090 sur un pur critère de valeur.*

**Mac Mini M4 + eGPU RTX 5090**

Cette combo précise revient constamment dans les questions. Elle ne fonctionne pas :

- **macOS sur Apple Silicon ne supporte pas CUDA sur des GPU externes, point final.** Le GPU NVIDIA connecté via Thunderbolt apparaîtrait pour la sortie display sur d'anciennes configs Mac Intel, mais sur Apple Silicon (M4), il n'y a aucun support de calcul GPU externe dans macOS
- **Pas de CUDA = pas de PyTorch, pas de ComfyUI, pas de vLLM, pas de Flux.2, pas de LTX-2** sous leurs formes standards. Il faudrait des alternatives Metal-only pour tout, qui n'existent pas pour la stack créative complète en 2026
- **La bande passante Thunderbolt 4 effective est ~5 Go/s**, même si CUDA fonctionnait, charger un modèle de 32 Go prendrait des minutes. Tout framework qui transfère des tenseurs entre CPU et eGPU serait paralysé
- **Coût total : ~4 000 €+** pour un système où la 5090 contribue un calcul nul

*Verdict : ne fonctionne pas. Le Mac Mini M4 est excellent comme plan de contrôle / routeur API toujours allumé utilisant MLX ou llama.cpp Metal, mais c'est un rôle de machine différent, pas un nœud de calcul GPU. La 5090 appartient à une machine Windows ou Linux.*

**Bande passante et mémoire Apple Silicon, toutes les puces M ne sont pas égales :**

| Puce | BW mémoire unifiée | Mémoire max | Idéal pour |
|------|-------------------|------------|---------|
| M1 Max | 400 Go/s | 64 Go | Petits modèles 7-13B, inférence légère |
| M4 Max | 410 Go/s | 128 Go | Jusqu'à ~34B BF16, 70B limité |
| M1 Ultra | 800 Go/s | 128 Go | 70B confortable |
| M4 Ultra | 819 Go/s | 192 Go | 70B+ confortable, 405B partiel |

M1 et M4 ne sont **pas** équivalents, le M4 Ultra (819 Go/s, 192 Go) est 2× plus rapide que le M4 Max (410 Go/s) pour l'inférence. Un Mac Studio M4 Max est souvent comparé à tort avec le Mac Studio M4 Ultra, ce sont des machines de classe différente. Toujours confirmer quelle puce en comparant des benchmarks.

**Mac Studio M4 Ultra (192 Go unifié)**

L'alternative la plus légitime pour un cas d'usage précis : faire tourner des modèles 405B en silence.

- **Ce qu'il fait uniquement bien :** 192 Go de mémoire unifiée à 819 Go/s, silencieux, 40W de consommation, fait tourner des modèles 405B+ en Q4 entièrement en mémoire, plug-and-play
- **Ce qu'il ne peut pas faire :**
  - Pas de CUDA. Pas de ComfyUI Flux.2, pas de LTX Video 2, pas d'entraînement LoRA PyTorch standard, pas de vLLM
  - L'inférence Apple MLX mûrit vite mais reste un sous-ensemble de l'écosystème CUDA. De nombreux modèles frontier-équivalents actuels et fine-tunes d'agents spécialisés n'ont aucun chemin MLX optimisé
  - ~30-40 tok/s sur 70B Q4 via MLX contre ~80-120 tok/s sur le 4x5090. La bande passante mémoire unifiée (819 Go/s) est plus basse par slot qu'une seule 5090 (1 792 Go/s)
  - À ~5 999 €, on obtient une seule machine sans chemin d'expansion

*Verdict : le bon choix si tout votre workflow est de l'inférence texte via MLX et que vous avez besoin du plus grand contexte possible en silence. Mauvais choix si une partie de votre workflow touche à ComfyUI, Flux.2, LTX-2, ou PyTorch standard.*

**M5 Ultra, attendu mi-2026 (estimation WWDC) :**
Apple a entièrement sauté le M4 Ultra (annulé). Le prochain Mac Studio sortira avec un M5 Ultra proposant :
- **80 cœurs GPU**, jusqu'à **256 Go de mémoire unifiée**, **~1 100 Go/s de bande passante**
- MLX seulement (pas de CUDA), même limitation que l'Apple Silicon actuel
- Prix estimé : ~6 000-8 000 €
- À 1 100 Go/s et 256 Go : l'inférence 70B approcherait ~50-60 tok/s, et les modèles 405B+ tiendraient confortablement

Le M5 Ultra est la seule alternative machine unique à venir qui se rapproche de la capacité VRAM du rig 4x5090, mais à 1/4 du coût et 1/50 de la consommation. Si CUDA n'est pas nécessaire (ComfyUI/Flux.2/Wan 2.2), le Mac Studio M5 Ultra sera convaincant à sa sortie.

**Threadripper PRO 7000 WRX90 + 4x RTX 5090**

La plateforme professionnelle adaptée. Détaillée dans la section Architecture.

- **x16 PCIe 5.0 complet par GPU**, pas de bifurcation, pas d'asymétrie
- **+3 500-4 500 €** par rapport à l'AM5 pour CPU + carte mère WRX90
- **Débit 5-15% meilleur** pour le parallélisme tensoriel multi-GPU
- **Compte pour l'entraînement**, la synchro sérieuse de gradients sur 4 cartes a besoin de la bande passante
- **Marginal pour l'inférence**, le goulot d'étranglement est la bande passante GDDR7 interne au GPU (1,8 To/s par carte), pas la communication PCIe inter-GPU

*Verdict : la bonne plateforme pour un rig d'entraînement de production ou un serveur IA commercial. Excessif pour de l'inférence + fine-tuning LoRA. Les 3 500 € économisés financent la 4e 5090.*

### NVIDIA DGX Spark (4 699 $) : la Méprise la Plus Courante

Le DGX Spark est le superordinateur IA grand public de NVIDIA : 128 Go de mémoire unifiée, stack CUDA complète, format desktop compact. Sur le papier, il ressemble à un concurrent direct.

En pratique, pour l'inférence 70B+, il ne l'est **pas** :

- **2,7 tokens/seconde en décodage sur Llama 70B**, c'est le benchmark publié par LMSYS (octobre 2025). Les 128 Go de mémoire unifiée tournent à ~273 Go/s. C'est le goulot d'étranglement. Même VRAM que le rig 4x5090, 30× plus lent en décodage 70B
- Le fine-tuning via QLoRA culmine à 5 079 tok/s pour Llama 3.3 70B, impressionnant, mais le rig 4x5090 a de la VRAM discrète avec un support d'entraînement CUDA complet
- Pour les **petits modèles** (8B-14B), le DGX Spark brille : compact, silencieux, configuration nulle, bon débit à son niveau cible
- **Deux DGX Spark reliés** (~9 400 $) font tourner Qwen3-235B à 11,7 tok/s, dans la même fourchette que le 4x5090 pour un coût plus élevé

*Verdict : la bonne machine pour un nœud d'inférence 8-30B compact toujours allumé. Mauvaise machine pour du 70B+ à vitesse interactive. Le plafond de bande passante mémoire unifiée est fondamental, pas contournable.*

**Comparaison de benchmarks :**

| Config | 70B Q4 tok/s | BW mémoire | Coût |
|--------|-------------|-----------|------|
| DGX Spark | ~2,7 tok/s | 273 Go/s unifié | 4 699 $ |
| Mac Studio M4 Ultra | ~30-40 tok/s | 819 Go/s unifié | ~5 999 € |
| **4x RTX 5090 (ce build)** | **~80-120 tok/s** | **1 792 Go/s × 4** | **~22 800 €** |

### NVIDIA RTX Pro 6000 Blackwell (96Go, ~8 000-9 000 €) : une Carte pour Tous les Gouverner ?

La RTX Pro 6000 Blackwell est un GPU de station de travail professionnelle avec 96 Go de GDDR7. Elle fait tenir un 70B FP8 sur une seule carte avec 26 Go restants pour le cache KV.

- **Ce qu'elle fait bien :** 70B FP8 sur une seule carte, ~8 400 tok/s sur 30B AWQ (proche de 4x RTX 4090 sur ces workloads), aucune complexité multi-GPU, mémoire ECC pour les sessions longues sans surveillance, 600W pour un seul slot
- **Pourquoi pas pour ce build :**
  - 8 000-9 000 € pour une carte = impossible de s'en offrir 4 (32 000 €+)
  - 96 Go < 128 Go, ne peut pas faire tourner un 70B FP16 ou un 405B confortable
  - Une seule carte = pas de LTX Video 2 + Qwen3 70B simultanément
  - **Pas de NVLink**, les configs multi-cartes font toujours face à des goulots PCIe comme les cartes grand public
  - 600W par carte × 4 = 2 400W avant l'overhead système, le même défi thermique que 4x 5090, pour 4× le coût GPU

*Verdict : convaincant pour une station de travail mono-carte où la complexité multi-GPU est indésirable. Pas le bon choix pour un serveur dédié multi-workload inférence + ComfyUI.*

### L'Arbre de Décision

```
Besoin de CUDA (ComfyUI / Flux.2 / LTX-2 / vLLM / PyTorch) ?
├── Non → Mac Studio M4 Ultra pour du 405B en silence
└── Oui
    ├── Budget < 8 000 € au total ?
    │   ├── < 5 000 € → 2x RTX 5090 (votre Nomad/Torrent actuel)
    │   └── < 8 000 € → 4x RTX 4090 si la 5090 est indisponible
    └── Budget ≥ 15 000 € (build complet) ?
        ├── Centré inférence → 4x RTX 5090 sur AM5 (ce build)
        └── Centré entraînement → 4x RTX 5090 sur Threadripper WRX90
```

Le 4x RTX 5090 sur AM5 se situe à l'intersection de la **VRAM CUDA maximale** et du **coût de plateforme minimal viable**. Ce n'est pas le système le plus élégant, mais c'est le bon pour ce cas d'usage.

---

## Coût vs. Cloud : Est-ce que Ça Vaut le Coup ?

### Tarification GPU Cloud (estimations mi-2026)

| Fournisseur | Config GPU | $/heure | Mensuel (24/7) |
|----------|-----------|--------|-----------------|
| RunPod | 4×A100 80Go | ~8-12 $/h | ~6 000-9 000 $ |
| Lambda | 4×H100 80Go | ~12-16 $/h | ~9 000-12 000 $ |
| vast.ai | 4×RTX 5090 | ~4-6 $/h | ~3 000-4 500 $ |
| Ce build | 4×RTX 5090 (possédé) | ~0,50 €/h d'électricité | ~360 €/mois |

### Analyse du Seuil de Rentabilité

Coût total du build : ~16 500 €. Électricité mensuelle à ~2 kW de moyenne (sous-volté, pas 24/7 pleine charge) : ~360 €/mois à 0,18 €/kWh.

Comparé à la location de 4x5090 sur vast.ai à ~5 $/h :
- Si le rig est utilisé **8 heures/jour** : coût cloud = ~1 200 $/mois contre votre électricité = ~120 €/mois
- **Seuil de rentabilité : ~14-16 mois** d'usage régulier
- Après ça, chaque mois d'usage est essentiellement gratuit (moins l'électricité)

Pour un cinéaste/chercheur qui fait tourner de l'inférence quotidiennement, fine-tune des LoRA chaque semaine, et a besoin d'un serveur API local permanent, ce build se rentabilise en un an et demi.

### Le Vrai Avantage : Disponibilité et Confidentialité

Au-delà du coût, posséder le hardware signifie :
- **Pas de file d'attente, pas de démarrage à froid**, vos modèles sont toujours chargés et chauds
- **Confidentialité totale des données**, rien ne sort de votre réseau
- **Pas de rate limit d'API**, lancer autant de requêtes que vos GPU peuvent gérer
- **Liberté d'expérimentation**, essayer des modèles bizarres, des quantifications custom, et des logiciels de pointe sans angoisse tarifaire à l'heure

---

---

## Nœud Complémentaire : Mac Mini M5 Pro ou Mac Studio M5

Le rig 4x5090 est une machine CUDA, bruyante, chaude, gourmande en énergie, et pas allumée 24/7 en permanence. Un Mac Mini ou Mac Studio M5 comble les manques : silencieux, 35-80W, toujours allumé, et capable d'une inférence MLX sérieuse pour tout ce qui n'a pas besoin de CUDA.

### Specs Supposées : Mi-2026 (estimation WWDC)

| Config | Mémoire unifiée | Bande passante | Prix estimé | Idéal pour |
|--------|---------------|-----------|------------|----------------|
| Mac Mini M5 (base) | 16-32 Go | 153 Go/s | ~699 € | Plan de contrôle uniquement, agents 7-14B |
| **Mac Mini M5 Pro 48 Go** | 48 Go | 307 Go/s | ~1 600 € | ★ Meilleur rapport qualité-prix, modèles 30B, 70B léger |
| **Mac Mini M5 Pro 64 Go** | 64 Go | 307 Go/s | ~2 100 € | ★★ Point idéal, 70B Q4 tient entièrement |
| Mac Studio M5 Max 128 Go | 128 Go | 614 Go/s | ~3 500 € | Vraie vitesse d'inférence 70B (~30-40 tok/s) |
| Mac Studio M5 Ultra 256 Go | 256 Go | 1 100 Go/s | ~6 500 € | 405B+ tient, cas d'usage de niche |

> Les specs sont supposées, Apple n'a annoncé ni Mac Mini M5 ni Mac Studio M5 en date d'avril 2026. WWDC attendu en juin 2026. [Rumeurs actuelles](https://www.macworld.com/article/2964754/2026-mac-mini-m5-pro-design-specs-release-date.html).

### La Recommandation

**Commencer avec le Mac Mini M5 Pro 48 Go (~1 600 €).**

- 307 Go/s, 48 Go de mémoire unifiée
- Fait tourner Qwen3 32B via MLX à ~25-35 tok/s, couvre 90% des tâches d'agent quotidiennes
- Le 70B Q4_K_M tient à 48 Go avec un cache KV minime ; plus lent (~12-18 tok/s) mais fonctionnel pour des tâches en arrière-plan
- Toujours allumé à ~35W, le rig 5090 peut dormir, le Mac Mini gère les appels Hermes Agent la nuit
- Le M5 Pro ajoute Thunderbolt 5 et Wi-Fi 7

Si le besoin de 70B à vraie vitesse ou de modèles 128B+ se fait sentir côté Mac : passer au Mac Studio M5 Max (128 Go, 614 Go/s). Le Mac Mini M5 Pro est le point d'entrée ; le Studio est le chemin de montée en gamme.

**Si le budget le permet dès le premier jour : Mac Studio M5 Max 128 Go** à ~3 500 € est réellement une machine différente, 614 Go/s signifie un 70B Q8 entièrement en mémoire unifiée à 30-40 tok/s. Comparable au rig 4x5090 sur le pur débit 70B (pas plus rapide, mais silencieux et 80W).

### Pourquoi Pas le Mac Studio M5 Ultra ?

L'Ultra (256 Go, ~6 500 €) n'est convaincant que si des modèles 405B+ sont nécessaires en local. Pour tout le reste, le Max 128 Go donne 85% de la capacité pour 55% du prix. L'Ultra est un achat d'affirmation pour 2026.

### EXO : Mutualiser Tout

Le vrai avantage émerge avec [EXO](https://github.com/exolabs/exo) : un framework qui mutualise la VRAM/mémoire unifiée entre appareils hétérogènes sur votre réseau Tailscale.

```bash
# Mac Mini M5 Pro : nœud EXO (backend MLX)
exo --node-id mac-mini --backend mlx

# Rig 4x5090 : nœud EXO (backend CUDA)
exo --node-id rig --backend cuda

# Depuis n'importe quelle machine : interroger le cluster mutualisé
curl http://localhost:52415/v1/chat/completions   -d '{"model": "llama-3.1-405b", "messages": [...]}'
# EXO répartit les couches sur les deux machines
```

**Ce qu'apporte la mutualisation :**
- 128 Go de VRAM (rig) + 48-64 Go de mémoire unifiée (Mac Mini) = jusqu'à ~192 Go distribués
- Le Mac Mini gère les couches natives MLX ; le rig gère les couches optimisées CUDA
- Des modèles plus gros deviennent atteignables sans acheter plus de GPU
- Chaque Mac supplémentaire ajouté plus tard élargit le pool

Chemin de montée en gamme : Mac Mini M5 Pro 48Go → ajouter un Mac Studio M5 Max → ajouter un second Mac Mini si besoin. Chaque achat élargit la capacité d'inférence distribuée sans toucher au rig CUDA.

---

## Ressources

- [Risers LINKUP AVA5 PCIe 5.0](https://linkup.one), validés pour la RTX 5090
- [Ollama](https://ollama.com), le runtime LLM local le plus simple
- [vLLM](https://github.com/vllm-project/vllm), inférence à haut débit avec parallélisme tensoriel
- [llama.cpp](https://github.com/ggml-org/llama.cpp), inférence GGUF flexible
- [NVIDIA System Management Interface](https://developer.nvidia.com/nvidia-system-management-interface), `nvidia-smi` pour le monitoring et la gestion de puissance
- [GPU-Z](https://www.techpowerup.com/gpuz/), vérifier la vitesse et la génération du lien PCIe

---

## Licence

Ce guide de montage est partagé à des fins éducatives. Les choix de hardware reflètent mes besoins spécifiques (cinéaste + chercheur IA) et mes contraintes de budget. Votre expérience peut varier.

Construit avec l'aide de Claude Code. Les erreurs sont miennes.

Par [Ismaël Joffroy Chandoutis](https://ismaeljoffroychandoutis.com).
