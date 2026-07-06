+++
title = "Cinquante compteurs et aucun diagnostic : lire les statistiques SRT comme un médecin"
date = 2026-07-06
description = "libsrt vous dit tout d'un flux qui meurt, sauf ce qui le tue. J'ai construit un petit outil qui échantillonne les statistiques toutes les 100 ms et nomme la maladie — puis j'ai laissé un lab plein de réseaux cassés réfuter ses trois premiers diagnostics."
+++

Mes billets précédents vivaient dans le monde HLS, où l'observabilité est un problème presque résolu : tout est HTTP, chaque maillon écrit une ligne de log, et quand quelque chose casse, on suit la requête. La contribution est le pays inverse. Un feed SRT est un unique flux UDP chiffré entre deux boîtiers, et quand il se dégrade, il n'y a pas de log d'accès à lire. Ce qu'on obtient à la place, c'est `srt_bstats()` : une structure C de plus de cinquante compteurs, mise à jour en direct, documentée laconiquement, et — c'est la partie exaspérante — tous *vrais* sans qu'aucun ne dise ce qui ne va pas.

Quiconque a exploité des liens SRT connaît l'appel au support. L'image a pixellisé à 14 h 32. Les compteurs montrent des pertes, des retransmissions, des drops, des paquets en retard, des niveaux de buffer, une estimation de RTT et une de bande passante. Lequel est la maladie et lesquels sont les symptômes ? Le réseau est-il mauvais ? La latence est-elle configurée trop bas ? Le lien est-il simplement trop petit ? Ces trois problèmes produisent des murs de chiffres superficiellement semblables et exigent trois remèdes *incompatibles*.

J'ai donc construit [`srt-doctor`](https://github.com/benjaminboisseau/srt-doctor), un petit CLI en Rust qui se place à l'une ou l'autre extrémité d'une session SRT, échantillonne la structure de statistiques complète toutes les 100 ms, et tente de nommer la condition au lieu d'imprimer le mur. Ce billet parle des trois maladies, des signatures statistiques qui les séparent, et — dans la tradition d'honnêteté de ce blog — des trois fois où le docteur lui-même a nommé avec assurance la mauvaise maladie sur un réseau contrôlé en labo, et a dû être corrigé par ses propres données.

## Trois maladies, trois remèdes incompatibles

**Maladie un : le réseau perd, et SRT compense.** Des paquets se perdent (`pktRcvLoss` grimpe), les NAK repartent, les retransmissions arrivent (`pktRcvRetrans` grimpe), et chaque trou est comblé avant son échéance de lecture : `pktRcvDrop` reste à zéro. C'est alarmant sur un dashboard et c'est en réalité le système qui fonctionne exactement comme prévu. Le remède : ne rien faire.

**Maladie deux : le budget de latence est trop petit.** Même réseau à pertes — mais la récupération perd la course. Une retransmission met au moins un aller-retour à arriver ; si `SRTO_LATENCY` n'achète pas ce temps (plus l'aller-retour suivant, quand la retransmission elle-même se perd), le récepteur abandonne et saute : `pktRcvDrop` grimpe, et le téléspectateur le voit. Rien n'a changé côté réseau par rapport à la maladie un. Le remède est un seul réglage, et aucune ingénierie réseau ne s'y substituera.

**Maladie trois : le lien ne peut pas porter le flux.** C'est la sournoise, parce que son symptôme le plus fort est le *silence* : souvent zéro perte signalée. Le buffer d'émission se remplit (`msSndBuf` grossit), et c'est l'émetteur lui-même qui jette des paquets jamais envoyés (`pktSndDrop`). Plus de latence — le réflexe — aggrave strictement les choses : cela donne juste plus de place à une file condamnée.

Vu du récepteur, les trois maladies donnent : des pertes avec retransmissions et sans drops ; des pertes avec retransmissions *et* drops ; et des drops sans **aucune** retransmission — cette dernière signature signifiant que les paquets manquants n'ont jamais été envoyés, pas qu'ils se sont perdus en vol. Trois colonnes différentes de la même structure. Une fois qu'on sait les lire comme un diagnostic différentiel, le mur de compteurs se réduit à une phrase.

## Un lab qui ne peut pas me mentir

Les anecdotes de terrain ne sont pas des preuves, et je n'écris pas sur les liens de mon employeur. Les données de ce billet viennent donc d'un banc reproductible qui tient dans un [script shell](https://github.com/benjaminboisseau/srt-doctor/blob/main/lab/run-scenarios.sh) : une paire veth dont une extrémité vit dans un namespace réseau Linux, `tc netem` qui dégrade le lien de façon contrôlée, ffmpeg qui produit un transport stream constant à 3,5 Mbps, et `srt-doctor` qui instrumente **les deux** extrémités de la session SRT — parce que, comme le montre la maladie trois, l'émetteur sait des choses que le récepteur ne peut que deviner.

L'expérience centrale est une paire de sessions sur un réseau dégradé *identique* — 80 ms de RTT, 2 % de pertes aléatoires — ne différant que par la latence configurée.

Avec `latency=500ms` (environ six allers-retours de budget) :

```
packets: 20681 unique | lost 398 (1.888%) | retransmitted 411 | dropped 0 (0.000%)

diagnosis: lossy but recovering
  - The link lost 398 packets (1.89%) but every one was retransmitted in time:
    the network is imperfect and the configuration is absorbing it.
  - configured 500 ms covers 4 retransmission round(s) at 1.89% loss — adequate
```

Avec `latency=80ms` — égale au RTT, donc aucune retransmission ne peut *jamais* arriver à temps :

```
packets: 20415 unique | lost 452 (2.166%) | retransmitted 448 | dropped 272 (1.332%)

diagnosis: LATENCY-STARVED
  - 272 packets were dropped unrecovered: the latency budget loses the race
    against loss recovery.
  - configured 80 ms is too small: 2.17% loss at 80.2 ms RTT (p95) needs
    ~482 ms to cover 4 retransmission round(s)

timeline (1 char = 1s): rLLLLLLLLLLLLLLLLLLLLLLLLLLLLrLLLLLLLLLLLLLLLLLLLLLLLLLLLLLL
```

Même réseau. Mêmes pertes. Zéro drop contre 272 — un artefact visible toutes les 200 ms pendant une minute. La différence entre « rien à corriger » et « irregardable » tenait à un entier dans un fichier de configuration.

## D'où vient le chiffre recommandé

La règle qu'on se transmet de régie en régie dit « latence SRT = 4 × RTT ». Elle marche jusqu'au jour où elle ne marche plus, parce qu'elle résume un raisonnement qui mérite d'être fait en entier — et la version complète tient sur un coin de nappe.

Suivez un paquet perdu. Le récepteur remarque un trou dans la numérotation et redemande le paquet (un NAK). La demande traverse le réseau dans un sens, la retransmission revient dans l'autre : à peu près un aller-retour, plus un peu de marge — disons 1,5 × RTT pour être prudent. La latence SRT, c'est simplement le temps que le récepteur s'accorde à attendre un paquet avant de continuer sans lui. Le budget de latence est donc en réalité un nombre de *tentatives* : combien de fois ce paquet peut-il être réclamé et réémis avant que son échéance n'expire ?

Une tentative suffit — si la retransmission arrive. Mais sur un lien qui perd 2 % des paquets, la retransmission elle-même meurt une fois sur cinquante. Il faut alors une deuxième tentative, parfois une troisième. Combien en prévoir ? Assez pour que la malchance répétée devienne invisible : je vise un paquet sur un million encore manquant après toutes les tentatives, soit environ un accroc visible par heure sur un flux typique. À 2 % de pertes, la probabilité que le même paquet meure quatre fois de suite est d'environ une chance sur six millions — sous la cible. Donc : quatre tentatives × 1,5 × 80 ms ≈ 480 ms. C'est exactement ce que l'outil recommandait plus haut, et ce que la session à 500 ms a confirmé empiriquement : 398 pertes, zéro drop.

La même arithmétique explique les deux façons dont le raccourci 4 × RTT échoue : à 0,1 % de pertes, deux tentatives suffisent et le 4 × RTT gaspille du délai qu'on pourrait rendre au téléspectateur ; à 5 %, il en faut cinq et le 4 × RTT ruine tranquillement votre soirée.

## La maladie qui ne signale aucune perte

La session « bande passante » plafonne le lien à 2,5 Mbps sous un flux à 3,6 Mbps. Le rapport côté émetteur est sans ambiguïté — et regardez la ligne RTT :

```
RTT min/mean/p95/max: 100.0/2888.2/3388.0/3449.2 ms
packets: 20688 unique | lost 0 (0.000%) | retransmitted 0 | dropped 0 | sender-dropped 19988

diagnosis: BANDWIDTH-STARVED
  - The sender dropped 19988 packets with a backed-up send buffer: the link
    cannot carry the stream rate. Reduce the encoder bitrate, raise MAXBW,
    or fix the link — extra latency will not help.
```

Zéro perte, zéro retransmission, et l'émetteur a jeté *la moitié du flux* avant qu'il ne touche le câble. Pendant ce temps le RTT — 20 ms sur ce lien au repos (le 100.0 de la colonne min est juste l'estimation de départ de libsrt, affichée avant qu'une première vraie mesure ait pu aboutir) — a gonflé jusqu'à 3,4 **secondes**, parce que la file du plafond de débit s'est remplie et que chaque ACK attendait derrière un quart de seconde de vidéo condamnée. Cette croissance de file, c'est le bufferbloat qui fait à un feed de contribution exactement ce qu'il fait à votre upload domestique pendant une sauvegarde, et c'est le meilleur indice du récepteur : des trous de séquence sans **aucune** retransmission pendant que le RTT explose signifient que les paquets manquants n'ont jamais été émis.

Le premier diagnostic de l'outil sur ce cas précis était exactement faux, et la manière vaut d'être avouée. Le récepteur voyait 31 % de « pertes » et calculait consciencieusement une recommandation de latence : *montez à environ 40 745 ms*. Quarante secondes. Le calcul était juste en interne et le remède aurait été un poison — la file se moque du temps que vous acceptez d'attendre des paquets que l'émetteur a déjà jetés. La version actuelle reconnaît la signature « zéro retransmission + RTT gonflé » et refuse tout conseil de latence, parce que le seul conseil honnête est : votre lien est trop petit.

## La maladie qui n'en est pas une

La meilleure trouvaille du lab est un accident. Je voulais un scénario « gigue » et j'ai donné à netem `delay 30ms 15ms` — un temps de trajet de 30 ms, plus ou moins 15 ms de variation aléatoire. La gigue, c'est exactement cela : non pas la durée du trajet, mais à quel point cette durée *varie* d'un paquet au suivant. Et des temps de trajet variables ont une conséquence facile à rater : ils battent les cartes. Prenez deux paquets émis à 5 ms d'écart. Si le premier tire 44 ms de délai et le second 18 ms, le second arrive 21 ms *avant* son aîné. Le réseau n'a rien perdu — il a mélangé.

Regardez maintenant depuis le fauteuil du récepteur. Les paquets arrivent numérotés 1, 2, 5 — et SRT en conclut, instantanément, que 3 et 4 manquent. Cette réaction instantanée est un choix de conception délibéré, et face à une vraie perte c'est la plus grande qualité de SRT : pas une milliseconde perdue avant de redemander le paquet manquant. Face au réordonnancement, elle devient une gâchette trop sensible : 3 et 4 n'ont jamais été perdus, ils sont 20 ms derrière, déjà en vol. Le NAK part quand même, l'émetteur renvoie les deux, et un instant plus tard le récepteur tient deux exemplaires de chacun — l'original, arrivé avec un retard élégant, et une retransmission dont personne n'avait besoin. Multipliez par chaque paquet mélangé sur le lien :

```
packets: 20688 unique | lost 15816 (43.327%) | retransmitted 15797 |
         dropped 0 (0.000%) | belated 15796

diagnosis: jittery
  - Jitter is reordering packets: 15816 "lost" packets came back belated (15796)
    with zero unrecovered drops. Not real loss — but the false NAKs cost 76%
    extra bandwidth in spurious retransmissions. Consider SRTO_LOSSMAXTTL
    to tolerate reordering.
```

Quarante-trois pour cent de pertes déclarées. Zéro drop. Une image parfaite — et un câble portant 6,4 Mbps pour un flux de 3,6 Mbps, aux trois quarts des retransmissions dont personne n'avait besoin. Ce mode de défaillance n'est pas une curiosité de labo : les liens cellulaires agrégés réordonnent par construction (deux moitiés du flux qui font la course sur deux opérateurs), le WiFi réordonne sous ses propres réémissions, et l'équilibrage de charge par paquet dans les cœurs de réseau le fait sous congestion. Il terrifie les dashboards, et il a un remède d'une ligne que presque personne ne déploie : `SRTO_LOSSMAXTTL N` dit au récepteur d'attendre d'avoir vu N paquets de plus avant de déclarer qu'un trou est une perte — de la tolérance au battage de cartes plutôt qu'une gâchette sensible. Il est livré désactivé (N = 0), parce que la tolérance coûte du temps de réaction quand la perte est réelle ; mais sur un lien connu pour réordonner, un petit N convertit 76 % de surcoût en zéro. Le compteur qui démasque toute la comédie est `pktRcvBelated` — des paquets « perdus » arrivés après qu'on les a abandonnés. Quand belated ≈ lost et drops = 0, vous n'avez pas un problème de pertes ; vous avez un problème d'ordre et une facture de bande passante.

Ce compteur `pktRcvBelated` m'a aussi offert ma deuxième confession : une version antérieure du diagnostic traitait *tout* paquet en retard comme une preuve de latence insuffisante. La session « pertes récupérées » a alors montré 24 paquets belated avec zéro drop — des retransmissions *dupliquées* en retard, qui ne coûtent rien — et l'outil a crié « latency-starved » sur un flux parfaitement sain. Les compteurs sont plus subtils qu'ils n'en ont l'air : belated avec drops signifie famine de latence ; belated sans drops signifie doublons ou réordonnancement. Une colonne, deux maladies, distinguées seulement par leur voisine.

## Ce qu'achète l'échantillonnage à 100 ms

Tout ce qui précède utilise des verdicts par fenêtres d'une seconde construites sur des échantillons de 100 ms, et la résolution compte : une moyenne à la minute de la session « latence trop courte » affiche « 2,2 % de pertes » — indiscernable du « 1,9 % » de la session saine. Le dégât vit dans la structure, pas dans la moyenne.

Une objection légitime : une rafale plus courte que 100 ms peut-elle se glisser *entre* deux échantillons ? Non — et la raison mérite d'être écrite noir sur blanc, parce que c'est elle qui rend le sondage viable. libsrt n'offre aucun flux d'événements pour ses statistiques ; la seule interface est `srt_bstats()`, qui recopie des *compteurs cumulatifs* que la bibliothèque met à jour à chaque paquet. L'échantillonnage ne rate donc jamais un événement : une rafale de 30 ms qui tue 15 paquets dépose les 15 dans l'échantillon qui la couvre. Ce que l'intervalle fixe, c'est la *précision temporelle* — à 100 ms, on sait quel dixième de seconde a souffert, pas quelle milliseconde. Et comme l'appel n'est rien de plus qu'une copie de structure sous verrou, le curseur se tourne quand il le faut : `--interval-ms 10` tourne sans effort (989 échantillons sur une session de dix secondes dans ce même lab) et résout la structure des rafales sous la durée d'une image. 100 ms est le défaut parce que cela sépare toutes les maladies de ce billet en produisant des journaux qu'un humain peut encore lire.

Le dernier scénario du lab montre ce que le défaut attrape déjà : un modèle de pertes de Gilbert-Elliott (rares états défaillants qui avalent ~10 paquets consécutifs) a produit une moyenne polie de 1,28 %, et la vue fine de l'outil :

```
  - loss is concentrated: only 4.3% of 100 ms samples saw any loss, but each
    hit lost ~10 packets — micro-bursts a coarse average would smooth away.
  - configured 300 ms is too small: 1.28% loss at 60.1 ms RTT (p95) needs
    ~361 ms to cover 4 retransmission round(s)
```

Ce verdict du conseiller a mérité son salaire : 300 ms configurées contre ~361 ms calculées nécessaires prédisent un flux qui tient *presque* — et de fait, 269 paquets perdus, 263 récupérés, et exactement 6 jetés. Un problème de marge que vous ne trouverez jamais dans une moyenne hebdomadaire. Les moyennes, c'est la façon dont un flux meurt sous un dashboard vert.

Tout est reproductible depuis le dépôt — l'outil, le script du lab, et les journaux JSONL bruts derrière chaque rapport cité. Si vous exploitez des liens de contribution et avez une histoire où les statistiques vous ont menti, je veux l'entendre.

*Outils : [srt-doctor](https://github.com/benjaminboisseau/srt-doctor) (Rust, MIT), libsrt 1.5.5, ffmpeg, tc/netem sur Alpine Linux.*
