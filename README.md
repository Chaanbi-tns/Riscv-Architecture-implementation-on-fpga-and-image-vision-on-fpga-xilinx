# Processeur RISC-V RV32I et SoC AXI en VHDL

Conception en VHDL d'un processeur **RISC-V RV32I monocycle** et de son **système sur puce (SoC)** : bus **AXI** à 5 canaux, mémoires ROM et RAM, UART et GPIO. Le projet est réalisé dans le cadre d'un bureau d'études (BE) de conception de SoC à l'ENIT (Génie Électrique), de l'unité arithmétique jusqu'au SoC complet.

![Architecture du SoC](figures/architecture_riscv.png)

---

## 1. Contenu du projet

| Étape | Contenu |
|---|---|
| Cœur RISC-V | ALU, banc de registres, décodeur, contrôleur, chemin de données monocycle |
| Interface mémoire | `BE_generator`, `write_data_manager`, `read_data_manager` (accès octet, demi-mot, mot) |
| SoC de base | Cœur + bus d'instructions et de données, ROM, RAM, fichiers d'initialisation `.mem` |
| SoC AXI | Cœur AXI multicycle, bus AXI 1 maître / 16 esclaves, ROM, RAM, UART, GPIO |
| Outils | Simulation ModelSim, synthèse Vivado, chaîne de compilation RISC-V (GCC) |

---

## 2. Le cœur RISC-V RV32I

- Jeu d'instructions **RV32I** : 37 instructions implémentées (10 de type R, 9 arithmétiques immédiates, 5 chargements, 3 rangements, 6 branchements, `jal`, `jalr`, `lui`, `auipc`).
- Architecture **monocycle** : une instruction par cycle d'horloge.
- Mémoire adressée à l'octet, **little-endian**.

### Chemin de données

Le contrôleur commande des multiplexeurs autour de l'ALU et du calcul de l'adresse suivante :

| Signal | Rôle | Valeurs |
|---|---|---|
| `PCrs1d_select` | base de l'adresse suivante | 0 = PC, 1 = rs1 |
| `imm4BJI_select` | décalage ajouté | 0 = 4, 1 = immB, 2 = immJ, 3 = immI |
| `A_select` | opérande A de l'ALU | 0 = rs1, 1 = PC |
| `B_select` | opérande B de l'ALU | 0 = rs2, 1 = 4, 2 = immI, 3 = immS, 4 = immU |
| `rdd_select` | donnée écrite dans rd | 0 = résultat ALU, 1 = donnée lue en mémoire |

Adresse suivante : `NPC = PCrs1d + imm4BJI`.

### Interface mémoire

Les accès à l'octet et au demi-mot passent par des **byte enables (BE)** :

| Accès | adresse[1:0] = 00 | 01 | 10 | 11 |
|---|---|---|---|---|
| Octet (SB, LB, LBU) | 0001 | 0010 | 0100 | 1000 |
| Demi-mot (SH, LH, LHU) | 0011 | non aligné | 1100 | non aligné |
| Mot (SW, LW) | 1111 | non aligné | non aligné | non aligné |

- `write_data_manager` réplique la donnée sur les 4 voies de l'octet.
- `read_data_manager` sélectionne les octets utiles, puis étend le signe (LB, LH) ou met des zéros (LBU, LHU).

---

## 3. Le SoC et sa carte mémoire

![Carte mémoire](figures/memory_map.png)

| Bloc | Rôle | Taille | Adresses |
|---|---|---|---|
| ROM0 | programme | 32 Kio (8192×32) | `0x0000_0000` – `0x0000_7FFF` |
| RAM0 | variables | 8 Kio (2048×32) | `0x8000_0000` – `0x8000_1FFF` |
| ROM1 | constantes | 8 Kio (2048×32) | `0x9000_0000` – `0x9000_1FFF` |
| UART0 | console série | 2 registres | `0xA000_0000` – `0xA000_0007` |
| GPIO0 | boutons et LEDs | 2 registres | `0xB000_0000` – `0xB000_0007` |

Le bus décode l'adresse et sélectionne l'esclave. Une adresse hors de ces plages est signalée par `illegal_address`.

Les mémoires ROM sont initialisées par des fichiers `.mem` (`ROM0.mem` pour le programme, `ROM1.mem` pour les constantes), générés depuis le programme assembleur par la chaîne de compilation (section « Programmes »).

---

## 4. Le bus AXI

![Bus AXI](figures/bus_axi.png)

Le bus suit le principe d'**AMBA AXI** : cinq canaux indépendants, chacun avec un signal `VALID` (émetteur) et un signal `READY` (récepteur).

| Canal | Sens | Contenu |
|---|---|---|
| AR | maître → esclave | adresse de lecture |
| R | esclave → maître | donnée lue et réponse (`resp`) |
| AW | maître → esclave | adresse d'écriture |
| W | maître → esclave | donnée à écrire et byte enables |
| B | esclave → maître | réponse d'écriture |

**Règle du handshake :** un transfert a lieu au front d'horloge où `VALID` et `READY` valent tous les deux 1. L'émetteur ne retire pas son information tant que le transfert n'a pas eu lieu.

**Réponses :** `00` OKAY, `10` SLVERR, `11` DECERR.

### Le cœur AXI (`RISCV32I_core_axi`)

Le cœur AXI exécute chaque instruction en plusieurs états : demande d'instruction (AR), réception (R), exécution, puis, pour un chargement ou un rangement, accès mémoire (AR/R ou AW/W/B). Le compteur de programme et le banc de registres ne sont mis à jour qu'à la fin de l'instruction.

Chaque esclave (ROM, RAM, UART, GPIO) est lui aussi décrit par une petite machine à états : lecture (attente de AR, puis envoi de R) et écriture (attente de AW, puis W, puis envoi de B).

---

## 5. Simulation

Compilation et simulation sous **ModelSim SE-64 2020.4**.

![Simulation ModelSim](figures/simulation_modelsim.png)

La courbe montre le cœur qui lit ses instructions dans la ROM : le signal `ARvalid` demande l'instruction, `Rvalid` et `Rready` marquent le transfert de la donnée.

---

## 6. Structure du dépôt

```
.
├── README.md
├── figures/                 images utilisées dans ce document
└── seance9/
    ├── vhdl/
    │   ├── riscv/           ALU, RegBank, decoder, controller, cœur, cœur AXI...
    │   ├── axi/             pack_axi_types, bus_axi_1x16
    │   ├── mem/             ROM, RAM (versions simples et AXI)
    │   ├── gpio/            gpio_axi
    │   ├── uart/            uart_rx, uart_tx, uart_axi
    │   └── soc/             soc, soc_fpga, testbench
    ├── modelsim/            scripts de compilation et de simulation
    └── prog/                programmes de test et fichiers .mem
```

---

## 7. Utilisation

### Compiler et simuler (ModelSim)

Dans la console ModelSim, depuis le dossier `seance9/modelsim` :

```tcl
do comp_soc.tcl      ;# compile toutes les bibliothèques dans l'ordre
do sim_soc.tcl       ;# lance la simulation
do wave_soc.tcl      ;# affiche les signaux principaux
```

Bibliothèques utilisées : `axi_lib` (types AXI et bus), `riscv_lib` (cœur), `mem_lib` (mémoires), `gpio_lib`, `uart_lib`, `soc_lib` (SoC et testbench).

### Programmes

Un programme assembleur est converti en fichiers `.mem` avec la chaîne GCC RISC-V (`riscv-none-elf-gcc`) :

```
riscv-none-elf-gcc -march=rv32i -mabi=ilp32 -nostdlib -nostartfiles prog.s -T link2.ld -o prog.elf
riscv-none-elf-objdump -D prog.elf > prog.dis
riscv-none-elf-objcopy -O verilog --only-section=.program --verilog-data-width=4 prog.elf ROM0.mem
riscv-none-elf-objcopy -O verilog --only-section=.constants --verilog-data-width=4 prog.elf ROM1.mem
```

Les sections du script d'édition de liens (`link2.ld`) doivent correspondre à la carte mémoire du SoC (section 3).

### Synthèse (Vivado)

Le projet Vivado cible une carte **Zybo (xc7z010)**. Le SoC est instancié dans `soc_fpga.vhd`, avec une horloge de 9,6 MHz générée par un `clk_wiz` et une liaison série à 9600 bauds. Les fichiers `ROM0.mem` et `ROM1.mem` doivent être ajoutés au projet comme fichiers mémoire, et leurs noms passés aux génériques `PROGRAM_FILE_NAME` et `CONSTANTS_FILE_NAME`.

---

## 8. État du projet

- Cœur RV32I monocycle, interface mémoire et SoC de base : validés en simulation ModelSim sur des programmes de test.
- SoC AXI (cœur AXI, bus, ROM, RAM, GPIO, UART) : décrit en VHDL et compilé ; la synthèse Vivado a été réalisée pour la carte Zybo.
- L'exécution sur la carte n'est pas documentée dans ce dépôt.

---

## 9. Outils

| Outil | Usage |
|---|---|
| ModelSim SE-64 2020.4 | compilation et simulation VHDL |
| Vivado 2025.2 | synthèse et implémentation (Zybo, xc7z010) |
| xPack `riscv-none-elf-gcc` | assemblage et édition de liens des programmes |
| Ripes | écriture et vérification des programmes assembleur |

---

## 10. Auteur et remerciements

Projet réalisé par **Chaanbi Hassen**, École Nationale d'Ingénieurs de Tunis (ENIT), Génie Électrique.

Support de cours et sujet de BE fournis par l'enseignant ; les schémas du cours sont cités à titre de référence.
