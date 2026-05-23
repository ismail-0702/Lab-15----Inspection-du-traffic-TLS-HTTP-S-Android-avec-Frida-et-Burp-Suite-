# Lab-15----Inspection-du-traffic-TLS-HTTP-S-Android-avec-Frida-et-Burp-Suite-

on utilise Burp Suite comme proxy pour intercepter le trafic, et Frida pour injecter un bypass SSL pinning si l'application refuse de laisser passer nos certificats. L'application cible est InsecureBankv2, une fausse application bancaire Android conçue pour être analysée exactement comme ça.

Lab réalisé dans un environnement de test contrôlé à des fins pédagogiques.


Environnement
ÉlémentDétailSystèmeWindows + PowerShellÉmulateurAndroid Studio Emulator 5554Frida17.8.0ProxyBurp Suite Community EditionBackendAndroLabServer (Python 2.7)Package ciblecom.android.insecurebankv2

Structure du projet
LAB15-SSL-Pinning-Frida/
│
├── README.md
├── hello.js                    ← test d'injection Frida
├── sslpin_bypass_universal.js  ← bypass SSL pinning

Étape 1 — Est-ce que Frida voit l'application ?
Comme toujours, on commence par s'assurer que tout communique bien avant d'aller plus loin :
bashfrida-ps -Uai
InsecureBankv2 doit apparaître dans la liste avec son PID et son package :
PID   Name            Identifier
7288  InsecureBankv2  com.android.insecurebankv2
<img width="1100" height="531" alt="Frida détecte InsecureBankv2" src="https://github.com/user-attachments/assets/04e3d59a-95be-4068-b7d0-0b37145eadf0" />

Étape 2 — Test d'injection rapide
Avant de lancer des scripts complexes, on vérifie que Frida peut bien s'injecter dans le processus :
bashfrida -U -f com.android.insecurebankv2 -l hello.js
Connected to Android Emulator 5554
Spawned `com.android.insecurebankv2`. Resuming main thread!
[+] Script injecté: Java.perform OK
<img width="1196" height="417" alt="Injection Frida confirmée" src="https://github.com/user-attachments/assets/fb05f33a-d898-46e2-9246-fec364711f1b" />
Tout fonctionne. On peut passer à la configuration du proxy.

Étape 3 — Configurer Burp Suite pour écouter le trafic Android
Dans Burp Suite, on configure un listener sur toutes les interfaces réseau :
Proxy > Proxy settings > Proxy listeners
Bind to port: 8080
Bind to address: All interfaces
Le "All interfaces" est important — ça permet à Burp d'écouter sur l'adresse IP locale que l'émulateur Android va utiliser pour envoyer son trafic.
<img width="977" height="664" alt="Configuration Burp Suite" src="https://github.com/user-attachments/assets/2fa2fa69-d194-43d4-bb1a-8094635ed064" />

Étape 4 — Pointer l'émulateur vers Burp
Dans l'émulateur Android, on configure le réseau Wi-Fi pour passer par Burp :
Settings > Network & Internet > Wi-Fi > AndroidWifi > Edit
Proxy: Manual
Proxy hostname: 192.168.11.130   ← adresse IP de la machine Windows
Proxy port: 8080
<img width="376" height="819" alt="Proxy configuré sur Android" src="https://github.com/user-attachments/assets/ba58bd2a-b827-4328-950a-72cc6e2a7d83" />

Étape 5 — Installer le certificat CA de Burp sur Android
Sans cette étape, Android va rejeter les connexions HTTPS interceptées par Burp parce qu'il ne reconnaît pas son certificat. On ouvre le navigateur de l'émulateur et on va sur :
http://burp
Ça télécharge le fichier cacert.der. On l'installe ensuite depuis :
Settings > Security > Encryption & credentials > Install a certificate > CA certificate
Android confirme avec : CA certificate installed.
<img width="395" height="822" alt="Téléchargement du certificat Burp" src="https://github.com/user-attachments/assets/781481a4-77b1-4869-8fa8-08deb7ed4561" />
<img width="393" height="810" alt="Certificat CA installé" src="https://github.com/user-attachments/assets/00a18a01-e580-49b0-ab59-7c008fc2b0bf" />

Étape 6 — Lancer le serveur backend
InsecureBankv2 a besoin d'un vrai serveur pour fonctionner. Le backend s'appelle AndroLabServer et tourne en Python 2.7 — attention, il ne fonctionne pas avec Python 3 (syntaxe incompatible).
bashcd D:\Desktop\Android-InsecureBankv2\AndroLabServer
py -2 app.py
The server is hosted on port: 8888
Si les dépendances ne sont pas installées pour Python 2, il faut les installer avec la bonne version :
bashpy -2 -m pip install -r requirements.txt

Si on ouvre http://192.168.11.130:8888 dans un navigateur et qu'on voit "Not Found" — c'est normal. Le serveur n'a pas de page d'accueil, il répond uniquement aux routes de l'application comme /login.

<img width="853" height="74" alt="Serveur backend lancé" src="https://github.com/user-attachments/assets/9d5a5fb1-f7c6-49d4-a192-a390d9aba024" />

Étape 7 — Connecter l'application au serveur
Dans InsecureBankv2, on renseigne l'adresse du backend :
Server IP: 192.168.11.130
Server Port: 8888
<img width="447" height="782" alt="Configuration serveur dans l'app" src="https://github.com/user-attachments/assets/cb192358-f696-4bab-abbd-63fc93c6c9d9" />
On se connecte avec les identifiants de test, et l'écran principal s'affiche — Transfer, View Statement, Change Password. L'application parle bien au serveur.
<img width="425" height="823" alt="Application connectée" src="https://github.com/user-attachments/assets/eff73fbd-3193-4b38-a7fe-71acb5d60832" />

Étape 8 — Observer les requêtes dans Burp Suite
Dans Burp Suite, sous Proxy > HTTP history, les requêtes envoyées par l'application commencent à apparaître. La plus intéressante, c'est celle-ci :
POST /login HTTP/1.1
Host: 192.168.11.130:8888
Content-Type: application/x-www-form-urlencoded
User-Agent: Apache-HttpClient/UNAVAILABLE (java 1.4)

username=jack&password=********
Les identifiants passent en clair dans le corps de la requête HTTP. Pas de chiffrement, pas d'encodage — tout est lisible directement dans Burp.
<img width="1872" height="385" alt="Trafic intercepté dans Burp" src="https://github.com/user-attachments/assets/886bddee-65a8-4d5d-b61f-cbaf4fa2ae06" />
<img width="1902" height="596" alt="Détail de la requête login" src="https://github.com/user-attachments/assets/2b4d4a3f-0a14-46f8-b3b3-9ee11175d2fe" />

Étape 9 — Bypass SSL pinning avec Frida
Pour les applications qui implémentent du SSL pinning — c'est-à-dire qui vérifient l'identité exacte du certificat serveur et refusent les intermédiaires comme Burp — on injecte un script de bypass :
bashfrida -U -f com.android.insecurebankv2 -l sslpin_bypass_universal.js
Le script intercepte les mécanismes TLS standards d'Android et les neutralise un par un :
[+] Universal SSL Pinning Bypass chargé
[+] Cible : com.android.insecurebankv2
[+] SSL bypass: SSLContext.init patché
[+] SSL bypass: TrustManagerImpl.verifyChain patché
[-] OkHttp CertificatePinner non trouvé
[+] SSL bypass: WebViewClient.onReceivedSslError patché
[+] Hooks SSL installés
Le message OkHttp CertificatePinner non trouvé n'est pas une erreur — ça veut juste dire qu'InsecureBankv2 n'utilise pas OkHttp pour le pinning. Les autres hooks sont actifs, et c'est ce qui compte.
<img width="1452" height="558" alt="Bypass SSL pinning actif" src="https://github.com/user-attachments/assets/9e619651-40ac-4ae1-aca3-304ddd13ccac" />
