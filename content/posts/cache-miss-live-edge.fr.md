+++
title = "Mesurer la pénalité de cache-miss au live edge HLS"
date = 2026-06-15
description = "Tout lecteur HLS court après le segment le plus récent — celui qui a le moins de chances d'être dans le cache du edge CDN. Un petit outil en Rust mesure la pénalité qui se cache de tous les tableaux de bord."
+++

Un lecteur HLS passe sa vie à courir après le même fichier : le segment le plus récent. Après chaque rechargement de playlist, il demande au CDN le segment publié quelques instants plus tôt, le joue, puis revient chercher le suivant. Or ce segment tout juste publié est aussi celui qui a le moins de chances de se trouver déjà dans le cache du edge CDN — et quand il n'y est pas, le lecteur paie une pénalité que presque aucun tableau de bord ne vous montrera.

J'ai écrit un petit outil en Rust, [`hls-probe`](https://github.com/benjaminboisseau/hls-probe), pour mesurer cette pénalité directement. Voici ce qu'il cherche, et pourquoi cela compte.

## Pourquoi le live edge est le pire cas pour un cache

Un nœud edge de CDN met les objets en cache à la demande. La première fois que quelqu'un, sur un point de présence donné, demande un objet, le edge n'a rien à servir : il relaie la requête en amont — vers un tier parent, et si nécessaire jusqu'à votre origine — récupère les octets, les met en cache, et alors seulement répond. C'est un *cache miss*. Toute requête ultérieure portant sur le même objet, jusqu'à son expiration, est un *cache hit* servi directement depuis le edge en quelques millisecondes.

Pour un catalogue de VOD, c'est un non-événement : les fichiers populaires sont « chauds » dans les secondes qui suivent leur publication et le restent. Le live edge casse cette hypothèse. Le segment que veut un lecteur a, par définition, quelques secondes d'âge. S'il est le premier spectateur à atteindre un nœud edge donné après la publication de ce segment, sa requête est un cache miss garanti, et le time-to-first-byte s'envole pendant que le edge remonte la requête jusqu'à l'origine.

La pénalité est réelle mais facile à manquer, car elle se cache des métriques que les exploitants surveillent d'ordinaire. Le taux d'offload du cache comme la latence moyenne du edge sont dominés par l'écrasante majorité des requêtes, qui sont des hits chauds. La poignée de premières requêtes froides — une par segment, par edge — se dilue dans la moyenne, alors même qu'elle frappe précisément les requêtes pour lesquelles les lecteurs sont les plus sensibles au temps.

## Une façon de la mesurer : fresh contre warmed

On ne peut pas observer une décision de cache depuis la seule lecture, mais on peut l'isoler avec une comparaison simple. Pour chaque segment nouvellement publié :

1. Le demander **immédiatement**, à l'instant où il apparaît dans la playlist — c'est la requête *fresh*, celle qu'un cache miss pénaliserait.
2. Attendre une target duration, puis redemander **la même URL** — entre-temps, votre propre première requête a réchauffé le edge : c'est la requête *warmed*, un hit quasi certain.

L'écart de time-to-first-byte entre les deux est la pénalité, mesurée sur l'objet même qu'un lecteur aurait été chercher. En répétant l'opération sur plusieurs segments, le bruit se moyenne. Et pour transformer la déduction en preuve, on lit les en-têtes de statut de cache du CDN sur les deux requêtes : une transition nette *miss → hit* confirme qu'on a bien mesuré du cache, et non de la gigue réseau.

`hls-probe --edge-test` automatise exactement cela. Il surveille la playlist live, attrape chaque segment dès sa publication, exécute la paire fresh/warmed et lit les en-têtes de cache.

## Un contrôle : une origine directe ne montre aucune pénalité

Avant de faire confiance à un résultat positif, il vaut la peine de vérifier que la méthode n'en fabrique pas. Pointé sur la démo live publique d'Unified Streaming — un packager servi par nginx, sans aucun cache CDN devant lui — l'outil rapporte honnêtement :

```
$ hls-probe --edge-test -p 6 https://demo.unified-streaming.com/k8s/live/stable/live.isml/.m3u8
  avg: fresh TTFB 53 ms, warmed 80 ms, penalty -27 ms
  reading: no meaningful fresh-segment penalty observed (cache already warm, or origin very close to the edge)
```

Les requêtes fresh et warmed sont statistiquement indiscernables ; la « pénalité » moyenne est négative, ce qui n'est que du bruit. Il n'y a pas de cache edge à manquer, donc pas de pénalité — et l'outil le dit, au lieu d'en inventer une. C'est exactement le résultat qu'on attend d'un témoin.

## La signature CDN : lire les en-têtes de cache

Sur un vrai CDN, la décision de cache n'est pas une supposition — le edge vous la donne, si vous la demandez. Les deux grands réseaux l'exposent directement.

CloudFront répond sur chaque objet :

```
X-Cache: Hit from cloudfront
X-Amz-Cf-Pop: FRA56-P4
Age: 7
```

L'en-tête `Age` est le révélateur : il compte les secondes écoulées depuis que le edge a récupéré l'objet auprès de l'origine. En balayant le même actif de test public, j'ai vu des objets allant de `Age: 73414` (chaud depuis vingt heures) jusqu'à `Age: 7` (tiré vers le edge il y a sept secondes) — la preuve que les objets entrent et sortent réellement du cache, plutôt que d'y résider éternellement.

Akamai est encore plus explicite, nommant le serveur edge et la décision :

```
X-Cache: TCP_MISS from a2-19-125-146.deploy.akamaitechnologies.com (AkamaiGHost/22.5.3-...)
```

`TCP_MISS` contre `TCP_HIT`, c'est la décision de cache en toutes lettres. Quand une requête fresh renvoie `TCP_MISS` autour de la centaine de millisecondes, et que le même segment, une durée de segment plus tard, renvoie `TCP_HIT` en quelques millisecondes, on observe la pénalité de cache-miss en flagrant délit — le premier spectateur par edge payant pour tous ceux qui suivent. Cette transition *miss → hit*, accrochée au segment fraîchement publié, est la signature que `--edge-test` est fait pour mettre au jour.

## Ce que la pénalité coûte au lecteur

Cent millisecondes paraissent anodines, jusqu'à ce qu'on les replace dans la timeline du lecteur. Celui-ci demande le segment le plus récent à l'instant même où la playlist avance, et il travaille contre un buffer qu'il cherche à garder petit pour la latence. Deux choses tournent mal quand cette requête est lente.

D'abord l'évidente : avec un buffer serré — tout ce qui est réglé pour la basse latence — un premier octet lent sur chaque nouveau segment grignote la marge de sécurité et, au pire, fait caler la lecture. Une pénalité de 100 ms sur un segment de 2 secondes, c'est cinq pour cent du budget envolés avant le premier octet média, et cela se répète à chaque segment.

Ensuite la sournoise : la logique de débit adaptatif (ABR) estime la bande passante disponible à partir de la vitesse de téléchargement des segments. Un cache miss fait arriver un segment lentement, pour une raison qui n'a rien à voir avec la connexion du spectateur. L'estimateur ABR ne peut pas faire la différence, interprète le téléchargement lent comme un réseau congestionné, et peut faire *descendre* le spectateur d'un palier de qualité — dégradant un flux dont le lien était pourtant parfaitement sain.

Aucune de ces deux défaillances n'apparaît comme une erreur CDN. Les octets ont été livrés ; le tableau de bord est au vert ; le spectateur a simplement reçu une expérience moins bonne que ce que le réseau permettait.

## Le correctif structurel

C'est l'un des problèmes que le Low-Latency HLS a été conçu pour traiter. Avec les rechargements de playlist bloquants et les preload hints, le lecteur peut indiquer au CDN *à l'avance* quelle partie arrive ensuite, et le edge peut maintenir la requête ouverte et streamer les octets à mesure que le packager les produit — réduisant l'aller-retour miss-puis-fetch à un unique échange pipeliné. La pénalité de cache-miss ne disparaît pas tant qu'elle cesse d'être un aller-retour séparé et sériel sur le chemin critique. Mesurer la pénalité d'abord vous dit combien il y a à gagner avant d'investir dans la migration.

## L'utiliser soi-même

`--edge-test` n'a besoin que d'une URL de playlist live :

```
hls-probe --edge-test -p 8 https://votre-cdn.example/live/master.m3u8
```

Deux mises en garde honnêtes. La mesure est statistique — une paire isolée peut n'être que de la gigue réseau, alors collectez-en plusieurs et fiez-vous aux transitions d'en-têtes de cache plutôt qu'à un chronométrage individuel. Et une sonde sur un canal de staging ou de test sans audience est un cas pessimiste : sans spectateurs réels, chaque segment fresh est un cache miss de première requête garanti, alors que sur un canal de production populaire, un autre spectateur réchauffe chaque edge en quelques millisecondes et la coalescence des requêtes épargne tous ceux qui suivent. Cet écart entre staging et production mérite lui-même d'être mesuré : il vous dit quelle part de votre latence de queue est réelle, et quelle part n'est qu'un artefact d'un test mené dans une pièce vide.

L'outil est open source, sous licence MIT : [github.com/benjaminboisseau/hls-probe](https://github.com/benjaminboisseau/hls-probe). Issues et pull requests bienvenues.
