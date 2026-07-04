+++
title = "Un flux, deux CDN : les validateurs que Google exige et que MediaPackage n'envoie pas"
date = 2026-07-04
description = "Les CDN de Google refusent de mettre en cache les objets de plus de 1 Mio si l'origine n'envoie pas ETag ou Last-Modified. AWS MediaPackage n'envoie ni l'un ni l'autre, parce que CloudFront ne les demande jamais. Croisez les clouds, et votre offload de cache s'effondre en silence."
+++

Voici un mode de défaillance sans message d'erreur. Prenez une origine HLS qui tourne sans histoire derrière un CDN depuis des années, placez un autre CDN devant — geste parfaitement banal dans un monde multi-CDN — et regardez votre offload de cache s'effondrer. La lecture fonctionne toujours. Tous les tableaux de bord sont au vert. Le seul symptôme, c'est que votre origine fait soudain le travail pour lequel le CDN a été embauché, et vous facture ce privilège.

La paire dont je veux parler, c'est AWS Elemental MediaPackage en origine et Google Cloud CDN en couche de diffusion, parce que leur désaccord est documenté, net, et invisible tant qu'on ne sait pas où regarder. Tout tient dans deux en-têtes de réponse : `ETag` et `Last-Modified`.

## Ce que Google exige avant de mettre en cache les gros objets

Les deux CDN de Google posent des conditions à la mise en cache des gros objets, et « gros » commence très exactement à 1 Mio.

[Cloud CDN](https://cloud.google.com/cdn/docs/caching) met en cache les objets de plus de 1 Mio par *remplissage par blocs* (chunked cache fill) : il les récupère auprès de l'origine par une série de requêtes de plages d'octets. Mais il ne s'y risque qu'une fois que l'origine a prouvé qu'on peut lui confier des requêtes Range, et la preuve est une réponse portant **tout** ceci :

- `Accept-Ranges: bytes`
- un `Content-Length` valide (ou un `Content-Range` sur un 206)
- `Last-Modified` **et** un `ETag` fort

Les validateurs ne sont pas décoratifs. Quand le cache assemble un objet à partir de plusieurs réponses partielles, l'`ETag` et le `Last-Modified` sont sa façon de vérifier que tous les morceaux viennent de la même version de la ressource — personne ne veut d'un segment dont le premier mégaoctet date d'un redémarrage d'encodeur et le second d'un autre. Une origine qui échoue au test est traitée comme ne supportant pas les plages d'octets du tout : pas de remplissage par blocs, taille cachable plafonnée à 10 Mio, et toute réponse de plus de 1 Mo est refusée à la mise en cache à la première requête.

[Media CDN](https://cloud.google.com/media-cdn/docs/caching) — celui que Google destine réellement à la vidéo — est plus strict et plus direct. Pour mettre en cache des réponses d'origine de plus de 1 Mio, l'origine doit envoyer un validateur : `Last-Modified` *ou* `ETag`, plus un `Date` et un `Content-Length` valides. Et comme les objets de plus de 1 Mio sont récupérés par requêtes de plages d'au plus 2 Mio, la documentation ajoute, sans prendre de gants : les objets de plus de 1 Mio *ne sont pas servis* si l'origine ne supporte pas les plages d'octets.

Faites maintenant l'arithmétique sur un segment vidéo. Quatre secondes de 1080p à 5 000 kbps, c'est environ 2,4 Mio. Même un segment de deux secondes à ce débit franchit la barre du Mio. Pratiquement chaque segment vidéo au-dessus de la SD vit précisément dans la zone où les règles de cache de Google ont des opinions. Les renditions audio passent sous la barre ; votre vidéo, non.

## Ce que MediaPackage envoie réellement

AWS Elemental MediaPackage est un packager just-in-time : il origine du HLS et du DASH à la demande, et il est explicitement conçu pour vivre derrière un CDN. Sa politique de cache est réellement bonne — [AWS documente](https://docs.aws.amazon.com/mediatailor/latest/ug/cdn-emp-caching.html) les valeurs de `Cache-Control` posées par ses origin endpoints, et elles sont exemplaires : quatorze jours sur les segments média (immuables une fois créés), une demi-durée de segment sur les manifestes (qui changent sans cesse), une seconde sur les playlists LL-HLS.

Ce que ses réponses ne portent pas, c'est un validateur. Pas d'`ETag`, pas de `Last-Modified`. Et la raison se lit dans cette même documentation : chaque mot du guide d'optimisation CDN d'AWS pour MediaPackage est écrit pour CloudFront, et **CloudFront n'a pas besoin de validateurs pour mettre en cache**. Il cache sur le seul `Cache-Control`, quelle que soit la taille de l'objet. Pour un packager just-in-time, l'omission se comprend même assez bien — quel serait le `Last-Modified` honnête d'un segment généré à la demande ? L'origine n'est pas cassée. Elle est spécialisée : une origine taillée sur mesure pour un CDN qui ne pose jamais la question que Google pose.

Assemblez les deux moitiés et le piège se referme. Derrière CloudFront : des TTL de quatorze jours, un offload superbe, affaire classée. Mettez Google Cloud CDN ou Media CDN devant le même endpoint — multi-CDN, migration cloud, contrat de diffusion qui change de mains — et chaque segment vidéo est un objet de plus de 1 Mio venant d'une origine sans validateur. Cloud CDN les refuse ; la documentation de Media CDN dit qu'ils ne sont pas servis du tout. Le flux qui se cachait parfaitement sur un cloud ne se cache plus sur l'autre, et rien nulle part ne signale d'erreur, parce que livrer chaque octet depuis l'origine n'est pas une erreur. C'est juste une facture.

## Rendre l'audit automatique

C'est une propriété vérifiable de l'extérieur en une requête, alors j'ai appris à [`hls-probe`](https://github.com/benjaminboisseau/hls-probe) à la vérifier. Depuis la v0.4, `--measure` rapporte les en-têtes de cache portés par les segments échantillonnés et avertit quand ils entrent en collision avec les règles documentées :

```
$ hls-probe --measure https://demo.unified-streaming.com/k8s/live/stable/live.isml/.m3u8
  origin headers: ETag "usp-84377CE5", Last-Modified Sat, 04 Jul 2026 13:26:37 GMT,
                  Accept-Ranges bytes, Cache-Control absent
  [warning] no Cache-Control on segments: CDNs in honor-origin mode
            will not cache them at all
```

Voilà un packager derrière un simple nginx : des validateurs gratuits (un serveur de fichiers peut difficilement les éviter), mais pas de `Cache-Control` — l'image exactement inversée du profil MediaPackage, et une autre façon de perdre son cache. Le flux de test d'Apple, servi par Akamai, arrive au contraire avec tout : `ETag` fort, `Last-Modified`, `Accept-Ranges: bytes`, `Cache-Control: max-age=600, public` — des segments de 5,7 Mio que n'importe quel CDN de n'importe quel cloud cachera avec plaisir. Et une origine au profil MediaPackage — `Cache-Control` présent, validateurs absents, segments de plus de 1 Mio — reçoit l'avertissement pour lequel cette fonctionnalité existe :

```
  [warning] segments up to 2456 KiB carry no ETag or Last-Modified: Google Media CDN
            will not cache objects over 1 MiB without a validator
  [warning] origin fails Google Cloud CDN's byte-range test (needs Accept-Ranges:
            bytes + strong ETag + Last-Modified): no chunked cache fill,
            cacheable size capped at 10 MiB
```

Le motif qui se dégage en sondant un peu partout est parlant : les origines qui servent des fichiers — nginx, Apache, S3 — émettent des validateurs sans qu'on leur demande, parce que le système de fichiers leur tend une date de modification et qu'un hash ne coûte rien. Les origines qui *synthétisent* du contenu à la volée n'ont rien de commode à mettre dans ces en-têtes, alors elles n'y mettent rien. L'absence passe donc inaperçue exactement aussi longtemps que votre CDN s'en moque — et devient porteuse le jour où vous changez.

## Que faire

Trois options honnêtes, dans l'ordre où je les essaierais.

**Interposer quelque chose qui ajoute les validateurs.** Un proxy léger entre le CDN de Google et MediaPackage peut les synthétiser. Pour les segments média, c'est sûr *parce qu'ils* sont immuables : un `ETag` fort dérivé de l'URL du segment et un `Last-Modified` posé à la première rencontre ne valideront jamais à tort un objet modifié, puisque l'objet ne se modifie jamais. Les manifestes sont l'inverse — ils changent sans cesse sous la même URL : exemptez-les (ils sont minuscules et à TTL court ; les règles du Mio ne les concernent pas) ou dérivez l'`ETag` du corps du manifeste. Un validateur incorrect est pire que pas de validateur ; c'est l'immuabilité qui rend celui-ci correct.

**Chaîner les CDN.** Gardez CloudFront directement devant MediaPackage — le couple pour lequel AWS a conçu l'ensemble — et faites du second CDN un client de CloudFront. Vous payez un double saut, mais les réponses de CloudFront vers le second CDN peuvent porter ce que celles de MediaPackage ne portent pas.

**Vérifier avant de signer.** Si vous évaluez un changement de CDN, un `hls-probe --measure` contre votre origine vous dit en dix secondes si vos segments portent ce que le CDN de destination exige. C'est la due diligence la moins chère de votre année.

La leçon dépasse ces deux fournisseurs : *cachable* n'est pas une propriété de votre flux, c'est une propriété de votre flux *et* du CDN qui le lit. Chaque paire origine-CDN renégocie le contrat, les clauses sont enfouies dans la documentation de cache de chaque CDN, et la clause pénale est silencieuse. Lisez les en-têtes que votre origine envoie réellement — ils sont à une requête de distance, et ce sont eux, le contrat.

L'outil est open source, sous licence MIT : [github.com/benjaminboisseau/hls-probe](https://github.com/benjaminboisseau/hls-probe).
