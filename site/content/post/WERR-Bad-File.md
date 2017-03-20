---
title: "WERR Bad File dans les replications Samba 4"
description: "Une erreur dans les réplications Samba 4."
tags: [ "linux","adminsys","samba4" ]
lastmod: 2016-11-08
date: "2016-11-08"
categories:
  - "Linux"
  - "Samba4"
  - "Sysadmin"
slug: "werr-bad-file-samba4"
autoThumbnailImage: false
thumbnailImagePosition: "top"
thumbnailImage: //res.cloudinary.com/alaintno/image/upload/c_scale,w_750/v1478608599/Samba_yghdql.png
metaAlignment: center
---

![samba4] (//res.cloudinary.com/alaintno/image/upload/c_scale,w_750/v1478608599/Samba_yghdql.png)

Samba 4 peut-être utilisé comme un contrôleur de domaine émulant les fonctionnalités d'un serveur Active Directory.

A ce titre, et ce comme un serveur AD, il s'occupe de synchroniser le contenu de l'annuaire Active Directory entre l'ensemble des Domain Controller déclarés.

Techniquement les synchronisations sont mise en place et déclenchées automatiquement :

  - Lors de la jonction d'un serveur au domaine et sa promotion au rôle de Domain Controller il est ajouté au serveurs à synchroniser;
  - Périodiquement les synchronisations sont déclenchées;

Les synchronisations peuvent également être déclenchées manuellement.

Il est possible de consulter l'état des synchronisation à l'aide de la commande suivante :

```bash
samba-tool drs showrepl
```

La sortie de cette commande vous renseigne sur les synchronisations sortantes "outbound" et entrantes "inbound", concernant les différentes partitions :

```bash
samba-tool drs showrepl
Default-First-Site-Name\DC1
DSA Options: 0x00000001
DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
DSA invocationId: XXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX

==== INBOUND NEIGHBORS ====

DC=ForestDnsZones,DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

CN=Schema,CN=Configuration,DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

CN=Configuration,DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

==== OUTBOUND NEIGHBORS ====

DC=ForestDnsZones,DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

CN=Schema,CN=Configuration,DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET

CN=Configuration,DC=example,DC=com
	Default-First-Site-Name\DC2 via RPC
		DSA object GUID: XXXXXXX-XXXXXXXXXXX-XXXXXXXXXXXXXX
		Last attempt @ Wed Nov  9 17:49:06 2016 CET was successful
		0 consecutive failure(s).
		Last success @ Wed Nov  9 17:49:06 2016 CET
```

Je ne détaillerais pas dans ce poste le rôle de chaque partition, mais il faut imaginer que l'on simule le comportement d'un AD, il est donc clair que cette synchronisation des partitions de l'annuaire est primordiale.

Et c'est la que l'erreur "WERR Bad File" vient ternir ce tableau.

J'ai eu à migrer un ensemble de serveur de Samba 3 vers Samba 4. Il respectait le standard de l'époque avec un design PDC/BDC (Primary Domain Controller et Backup Domain Controller).

Une fois la migration terminée (que je détaillerais dans un prochain post) plusieurs serveurs affichaient des erreurs de synchronisation :
  - WERR Bad Netpath
  - WERR Bad file

L'erreur 'Bad Netpath' fut simple à corriger : un reboot du serveur.

L'erreur 'Bad File' quand à elle m'a donner un peu plus de mal.

La procédure fut de sortir le serveur du domaine ( en le décomissionnant ), de nettoyer les bases de samba puis le joindre à nouveau au domaine.

Je n'ai pas eu à me soucier des rôles FSMO pour ce serveur car il sont tous détenus par le serveur "maitre".

Sur le serveur provoquant l'erreur :
```bash
samba-tool domain demote -Uadministrator
rm -rf /var/lib/samba/*
samba-tool domain join domain.example.com DC -U"DOMAIN\administrator" --dns-backend=SAMBA_INTERNAL
```

Après quelques instants le serveur se synchronise automatiquement et reçoit l'ensemble des partitions correctement !

Il suffit de vérifier dans ``` /var/lib/samba/private/sam.ldb.d/ ``` pour constater que la date de modification des fichiers correspond à celles du "maitre".