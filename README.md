# Rapport d'Exploitation : Challenge Android Firestorm

## 1. Introduction
*   **Cible :** `FireStorm.apk`
*   **Niveau :** Medium
*   **Objectif :** Analyser l'application, extraire des identifiants confidentiels et forcer l'exécution d'une fonction d'authentification cachée pour récupérer le flag final.
*   **Méthodologie :** Approche "Black Box" (Analyse à partir de zéro, sans connaissance préalable du code source).

---

## 2. Phase 1 : Analyse du Manifeste (Point de départ)
Conformément à la méthodologie standard d'audit Android, le premier fichier analysé est le `AndroidManifest.xml`. Ce fichier sert de carte d'identité à l'application et guide toute la stratégie d'attaque.

### A. Identification de la cible
Le manifeste révèle le nom du paquet : `package="com.pwnsec.firestorm"`. Cet identifiant est l'élément de base requis pour toutes les étapes de communication via ADB et Frida.

### B. Analyse de la surface d'attaque
L'examen des permissions a révélé la présence de :
```xml
<uses-permission android:name="android.permission.INTERNET"/>
```

Déduction du hacker : La présence de cette permission confirme que l'application échange des données avec un serveur distant. Cela oriente immédiatement mes recherches vers des configurations réseau ou des clés d'API.

### C. Détermination du point d'entrée
Le manifeste définit la MainActivity comme l'activité de démarrage (Launcher) :
````Xml
<activity android:name=".MainActivity" ...>
````
C'est sur cette classe précise que j'ai concentré l'analyse du code source pour comprendre l'initialisation de l'application.

## 3. Phase 2 : Extraction de données (Analyse des Ressources)
Suite à la confirmation de l'utilisation du réseau dans le Manifeste, j'ai exploré les ressources statiques. Un vecteur courant de fuite d'information est le stockage de constantes dans res/values/strings.xml.
L'inspection de ce fichier a permis de découvrir une configuration Firebase complète :

firebase_api_key : Clé d'accès à l'API Google.
firebase_email : Identifiant de service (TK757567@pwnsec.xyz).
firebase_database_url : URL de la base de données Realtime Database.

## 4. Phase 3 : Instrumentation Dynamique (Hooking avec Frida)
Pour contourner l'absence d'appel de la fonction génératrice, j'ai utilisé Frida pour injecter un agent JavaScript en mémoire et forcer l'appel système de cette méthode.
### A. Mise en place du pont de communication
Le serveur Frida a été déployé sur l'émulateur via ADB :
````Bash
adb push frida-server /data/local/tmp/
adb shell "su -c 'chmod +x /data/local/tmp/frida-server'"
````
## B. Injection et exfiltration
L'application a été démarrée sous le contrôle de Frida pour injecter le script de hooking :
````Bash
frida -U -f com.pwnsec.firestorm -l frida_firestorm.js
````
Résultat : Le script a intercepté l'instance de la classe et a retourné la valeur générée en temps réel.
Mot de passe extrait : C7_dotpsC7t7f_._In_i.IdttpaofoaIIdiDnndIfC

## 5. Phase 4 : Exploitation et Récupération du Flag
La phase finale a consisté à simuler un client légitime via un script Python (get_flag.py) pour interroger l'API Firebase.
Processus :
Authentification : Envoi de l'email (extrait des ressources) et du mot de passe (extrait via Frida) à l'API Google Identity.
Obtention du Token : Récupération d'un idToken sécurisé.
Lecture de la Database : Utilisation du token pour lire les données protégées de la Realtime Database.
Flag récupéré : <img width="873" height="94" alt="Capture d&#39;écran 2026-04-15 163316" src="https://github.com/user-attachments/assets/8a04e4f8-e2a3-4807-a636-4696893645ad" />


## 7. Conclusion et Recommandations
Cette intrusion met en évidence plusieurs vulnérabilités critiques :
### Fuite de données dans les ressources : Les clés API et identifiants ne devraient jamais être stockés en clair dans strings.xml.
### Code Mort (Dead Code) : Laisser des fonctions de debug ou de test dans une version de production offre des points d'entrée aux attaquants.
### Limites de l'offuscation : L'offuscation des noms n'est pas une mesure de sécurité suffisante contre l'analyse dynamique (Instrumentation).
