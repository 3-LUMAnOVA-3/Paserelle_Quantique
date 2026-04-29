

# MISE À JOUR ARCHITECTURALE : SHADOW-GATE v2.0

Cette documentation trace l'évolution de la Passerelle Quantique suite aux audits de sécurité matérielle (Ring 0) et aux optimisations réseau récentes. L'architecture a basculé vers un modèle "Zero-Footprint" pour contourner les limitations des noyaux embarqués d'usine.

## 1. INFRASTRUCTURE AMNÉSIQUE (STATELESS RAM-OS)
Suite à une analyse de sécurité (Reverse Engineering du noyau Rockchip), l'absence des modules `dm_mod` et `fuse` a été identifiée. La stratégie de chiffrement a donc pivoté vers une **Infrastructure Amnésique**.
* **Le Concept :** L'OS et les microservices (Neural-DPI) tournent intégralement dans la mémoire vive (RAM) volatile.
* **Le Scellement :** À l'extinction, la RAM est compressée et chiffrée à la volée (OpenSSL AES-256-CBC) en un bloc monolithique inerte (`shadow_vault.enc`) déposé sur une partition SD non-journalisée (EXT4).
* **Anti-Forensics :** En cas de coupure de courant ou de vol physique, la RAM se vide instantanément. La carte SD ne contient aucune arborescence lisible.

## 2. HARDWARE ROOT OF TRUST & FORGE QUANTIQUE (NAND)
Pour garantir l'intégrité du déchiffrement du sas en RAM, nous avons établi une Racine de Confiance Matérielle (Hardware Root of Trust).
* **Clé Post-Quantique (Air-Gapped) :** La clé de déchiffrement maître n'est pas générée sur l'appareil Edge. Elle est forgée à 100% avec des algorithmes Post-Quantiques sur un hyperviseur sécurisé (Proxmox Cluster), puis flashée de manière permanente dans la mémoire NAND interne de 256 Mo du Luckfox.
* **Extraction Sécurisée :** Le script de démarrage (S99shadowgate) extrait cette clé en mémoire volatile uniquement lors du Boot-Sequence, assurant qu'elle ne touche jamais le stockage amovible.

## 3. OPTIMISATION QOS & CORRECTION KERNEL (UPSTREAM)
La passerelle intègre désormais un patch réseau majeur pour garantir une latence proche de zéro lors de l'inspection des paquets.
* **Patch QoS (Quality of Service) :** Une correction algorithmique a été développée et déployée (approuvée et mergée upstream sur GitHub) pour corriger les erreurs d'allocation réseau sur les processeurs limités (<= 16 Go de RAM). 
* **Impact :** Cette mise à jour permet au Luckfox d'optimiser ses tampons réseau (Network Buffers) sans saturer sa mémoire, garantissant une interception fluide pour notre module de sécurité.

## 4. SÉCURITÉ TEMPORELLE & ACCÉLÉRATION NPU
* **Ancrage RTC :** L'ajout d'une batterie matérielle (RTC) permet au système de s'éveiller avec la temporalité exacte (`hwclock -s`). Cela immunise la passerelle contre les attaques par rejeu (Replay Attacks) et garantit la validité des certificats mTLS Post-Quantiques, même après des mois d'isolation réseau.
* **NPU Overdrive (Zero-Day Exploit) :** Exploitation d'un DebugFS constructeur laissé ouvert (`WRITEABLE clk DebugFS`) pour overclocker dynamiquement le NPU (Neural Processing Unit). Ce surrégime permet d'absorber le calcul d'entropie basé sur le bruit thermique et d'alimenter nos filtres eBPF/XDP Ring 0 en temps réel.
