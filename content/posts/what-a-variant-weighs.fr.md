+++
title = "Ce que pèse vraiment une variante : BANDWIDTH face au réseau"
date = 2026-07-02
description = "L'attribut BANDWIDTH d'une master playlist HLS est une déclaration, pas une mesure. Télécharger de vrais segments et les peser ne coûte rien — et cela a surpris mon propre outil en train d'annoncer 780 Mbps pour un flux à 8 Mbps."
+++

Toute master playlist HLS s'ouvre sur une série de promesses. Chaque ligne `EXT-X-STREAM-INF` déclare un `BANDWIDTH` — les bits par seconde qu'un lecteur doit budgéter pour cette variante — et le lecteur la croit sur parole : il choisit le palier le plus haut dont la bande passante déclarée tient sous son estimation de débit. Toute l'échelle du débit adaptatif repose sur l'honnêteté de ces chiffres.

Mais `BANDWIDTH` est une déclaration, saisie dans la configuration d'un packager par quelqu'un, quelque part, il y a peut-être des années. La vérité est sur le réseau : les segments eux-mêmes, chacun un nombre connu d'octets couvrant un nombre connu de secondes. Diviser l'un par l'autre n'a rien de sophistiqué. C'est pourtant étonnamment rare — j'ai donc ajouté un mode `--measure` à [`hls-probe`](https://github.com/benjaminboisseau/hls-probe) qui télécharge les derniers segments d'une variante et les pèse.

Ce billet raconte ce que cette pesée vous apprend, et l'instant où mon propre outil m'a annoncé qu'un flux à 8 Mbps tournait à 780 Mbps. On y viendra. C'est instructif.

## Trois choses qu'un segment téléchargé vous dit

Une fois les vrais octets d'un segment en main, trois faits tombent gratuitement.

**Le conteneur.** Les premiers octets d'un segment le trahissent. Un segment MPEG-TS est un train de paquets de 188 octets, chacun ouvert par l'octet de synchronisation `0x47` — vérifiez l'octet 0 et l'octet 188, et vous savez. Un segment fMP4/CMAF commence par une box ISO-BMFF : les octets 4 à 8 se lisent `ftyp`, `styp`, `moof` ou consorts. Aucun framework de parsing nécessaire ; c'est l'équivalent, pour les formats de fichiers, de lire l'étiquette sur la boîte de conserve :

```
measure: 3 segment(s) sampled, container MPEG-TS
```

**Le débit réel.** Les bits utiles du segment divisés par la durée `EXTINF` déclarée donnent le débit média effectif — la chose même que `BANDWIDTH` prétend borner. La RFC 8216 §4.3.4.2 est précise sur le contrat : `BANDWIDTH` doit être une borne supérieure du débit crête par segment de la variante. Qu'un segment échantillonné la dépasse, et ce n'est pas une erreur d'arrondi : c'est une violation de conformité avec conséquences. Un lecteur qui a budgété le chiffre déclaré rencontre un segment qu'il n'avait pas prévu, et c'est son buffer qui règle l'addition.

**Le comportement de livraison.** Le time-to-first-byte et le débit de téléchargement complet par segment, qui disent comment le chemin se comporte à l'instant T — et qui ont nourri le [travail sur les cache-miss](/fr/posts/cache-miss-live-edge/) du billet précédent.

Pointée sur la démo live publique d'Unified Streaming, la variante la plus haute déclare un `BANDWIDTH` de 1 316 kbps (moyenne 1 196 kbps), et l'outil rapporte :

```
measure: 3 segment(s) sampled, container MPEG-TS
  measured bitrate 653 kbps (peak segment 653 kbps), avg TTFB 31 ms
  153 KiB in 42 ms (ttfb 27 ms) -> 29903 kbps throughput
```

Un contenu mesuré confortablement sous le plafond déclaré — la bonne relation, puisque la déclaration est une borne de crête et que ce moment précis du contenu était peu exigeant. Rien de spectaculaire. Le spectaculaire est venu plus tard.

## La confession : 780 Mbps de pure fiction

Pour exercer la détection fMP4, j'ai pointé `--measure` sur le flux d'exemple public d'Apple — le vénérable BipBop, dans son incarnation fMP4. Le palier le plus haut de cette échelle déclare environ 8 224 kbps. Mon outil a rapporté, avec un aplomb total :

```
measured bitrate 780666 kbps (peak segment 780666 kbps)
  571777 KiB in 2235 ms -> 2095390 kbps throughput  main.mp4
[warning] peak segment bitrate 780666 kbps exceeds declared BANDWIDTH 8224 kbps
```

780 Mbps. Par segment : 571 777 KiB — soit 558 mégaoctets — pour six secondes d'images de test dignes de la définition standard. Mon outil venait de découvrir l'encodeur le plus inefficace de l'histoire, et il a même dressé un avertissement de conformité contre Apple pour l'occasion. L'accusation n'a rien à ajouter.

Le vrai coupable, évidemment, c'était moi. Ce flux utilise l'**adressage par plages d'octets** : au lieu d'un fichier par segment, toute la variante vit dans un unique `main.mp4`, et chaque entrée de la playlist y découpe sa tranche avec un tag `EXT-X-BYTERANGE` :

```
#EXTINF:6.00000,
#EXT-X-BYTERANGE:5874288@721
main.mp4
```

Lire : ce segment fait 5 874 288 octets à partir de l'offset 721 de `main.mp4`. Mon outil ignorait le tag et téléchargeait l'URI — le film entier de 558 Mo, une fois par segment échantillonné, trois fois de suite. Puis il divisait 558 Mo par six secondes et rapportait le résultat sans ciller. Déchets en entrée, déchets assurés en sortie : un outil de mesure bogué n'échoue pas bruyamment, il ment avec une excellente tenue.

## Le correctif, et le piège dans le correctif

Le correctif évident consiste à honorer le tag : envoyer un en-tête HTTP `Range: bytes=721-5875008` et ne récupérer que la tranche. La subtilité se cache dans la RFC 8216 §4.3.2.2 : l'offset de `EXT-X-BYTERANGE:<n>[@<o>]` est *optionnel*. Quand il est omis, la sous-plage commence **là où la sous-plage précédente de la même URI s'est arrêtée**. La playlist n'est pas une liste de faits indépendants ; c'est un registre courant, qu'il faut rejouer depuis le début pour savoir où commence réellement une entrée sans offset. Ne résolvez que la queue de la playlist — les segments les plus récents, précisément ceux qu'on veut échantillonner — et chaque offset omis retombe silencieusement à zéro, ce qui vous ramène dans le commerce de la fiction, avec des étapes en plus.

Le résolveur parcourt donc toute la playlist dans l'ordre, en tenant un curseur par URI, et alors seulement remet aux derniers segments leurs plages absolues. Après correction, sur le même flux Apple :

```
measure: 3 segment(s) sampled, container fMP4/CMAF (init segment present)
  measured bitrate 7803 kbps (peak segment 7806 kbps), avg TTFB 24 ms
  5717 KiB in 104 ms (ttfb 64 ms)
```

7 803 kbps mesurés contre un plafond déclaré de 8 224 kbps. Un contenu juste sous sa borne de crête déclarée — exactement la relation que prescrit la RFC, récupérée d'une erreur d'un facteur cent en relisant une section de la spec avec plus d'attention.

## Pourquoi peser, au fond

Parce que les chiffres déclarés pilotent des décisions réelles. Un `BANDWIDTH` sous-évalué pousse les lecteurs à s'engager sur un palier que le réseau ne tient pas ; surévalué, il gaspille une qualité que la ligne du spectateur aurait portée. Un segment crête qui dépasse réellement sa déclaration — l'avertissement que mon outil a levé à tort, mais qu'il peut désormais lever à raison — est le genre de défaut qui se manifeste en rebuffering intermittent sur le palier le plus haut et résiste au diagnostic, précisément parce que tous les tableaux de bord affichent l'échelle *déclarée*, pas celle qui est livrée.

Et l'épisode des plages d'octets porte sa propre morale, qui me vaut plus que la fonctionnalité : une mesure ne vaut que la requête qui la produit. Les chiffres avaient l'air assez plausibles pour être publiés, jusqu'au moment où ils sont devenus absurdes — et il a fallu un flux que je n'avais pas fabriqué, avec un schéma d'adressage que mon chemin nominal n'avait jamais croisé, pour l'exposer. Testez vos instruments sur du matériel que vous n'avez pas produit. Ils vous remercieront en vous ridiculisant tôt, et en privé — c'est le bon timing.

## L'utiliser soi-même

```
hls-probe --measure -p 5 https://example.com/live/master.m3u8
```

`-p` choisit combien de segments récents échantillonner par variante ; ajoutez `--json` pour une sortie machine (débits en bps bruts là-bas ; la sortie humaine parle en kbps, comme il se doit). Les playlists à plages d'octets, les init segments et les deux familles de conteneurs sont pris en charge. L'outil est open source, sous licence MIT : [github.com/benjaminboisseau/hls-probe](https://github.com/benjaminboisseau/hls-probe). Issues et pull requests bienvenues — surtout celles qui font rougir.
