# A. Base64
 ## 1. Génération du fichier binaire (100 Ko)
 dd if=/dev/urandom of=data.bin bs=1k count=100

 ls -lh data.bin
 
 ## 2. Encodage en Base64
 openssl base64 -e -in data.bin -out data.b64

 cat data.b64

 ls -lh data.bin data.b64
 
 ## 3. Décodage
 openssl base64 -d -in data.b64 -out data_restored.bin
 
 diff -s data.bin data_restored.bin

## 4. Réponses rapides

Base64 est-il un chiffrement ?
Non, c’est un encodage, il ne repose sur aucun secret, on peut revenir facilement au binaire original.
​

Pourquoi la taille change ?
Base64 représente 3 octets binaires par 4 caractères texte, donc le fichier grossit.
​

Pourcentage d’augmentation :
En gros, environ +33% de taille (4/3 ≈ 1,33).
​

Méthode rigoureuse pour vérifier l’égalité :
diff -s, ou un hash comme sha256sum data.bin data_restored.bin

# B. Chiffrement symétrique – AES
## 1. Création du message

nano confidentiel.txt

## 2. Chiffrement (AES-256, sel, PBKDF2, hash sécurisé)

openssl enc -aes-256-cbc -e -salt -pbkdf2 -md sha256 -in confidentiel.txt -out 

confidentiel.enc

## 3.Déchiffrement
openssl enc -aes-256-cbc -d -salt -pbkdf2 -md sha256 -in confidentiel.enc -out 

confidentiel_dechiffre.txt

diff -s confidentiel.txt confidentiel_dechiffre.txt

## 4. Analyse (rechiffrement)

openssl enc -aes-256-cbc -e -salt -pbkdf2 -md sha256 -in confidentiel.txt -out 

confidentiel2.enc

ls -lh confidentiel*.enc

diff confidentiel.enc confidentiel2.enc

## 5. Réponses rapides
Pourquoi les deux fichiers chiffrés sont différents ?
Chaque chiffrement utilise un sel aléatoire (et IV), donc le résultat change même avec le même mot de passe.
​

Rôle du sel ?
Empêcher les rainbow tables, rendre chaque chiffrement unique, compliquer les attaques par dictionnaire.

Si une option change au déchiffrement ?
Si algorithme, sel, dérivation ou hash ne correspondent pas, le déchiffrement échoue ou donne des données illisibles.
​

Pourquoi PBKDF2 ?
Pour dériver une clé forte à partir d’un mot de passe, en rendant les attaques par force brute beaucoup plus lentes (itérations).

Différence encodage / chiffrement ?
Encodage = transformation réversible sans secret (Base64). Chiffrement = transformation qui nécessite une clé pour revenir au clair.

# C. Cryptographie asymétrique – RSA
## 1. Génération de clés (clé privée protégée)
Générer la clé privée, avec chiffrement (mot de passe demandé) :

openssl genrsa -aes256 -out rsa_private.pem 2048

Exporter la clé publique :

openssl rsa -in rsa_private.pem -pubout -out rsa_public.pem

Afficher paramètres détaillés :

openssl rsa -in rsa_private.pem -text -noout

openssl rsa -in rsa_public.pem -pubin -text -noout

## 2. Chiffrement asymétrique d’un fichier

Créer le fichier :

nano secret.txt
Chiffrer avec la clé publique :

openssl pkeyutl -encrypt -in secret.txt-inkey rsa_public.pem -pubin -out secret.enc

Déchiffrer avec la clé privée :

openssl pkeyutl -decrypt -in secret.enc-inkey rsa_private.pem -out secret_dechiffre.txt

diff -s secret.txt secret_dechiffre.txt


## 3. Réponses rapides

- Pourquoi la clé privée ne doit jamais être partagée ?  
  Elle permet de déchiffrer les données et de signer, donc si quelqu’un l’a, il peut se faire passer pour toi et lire tout. 
- Pourquoi RSA n’est pas adapté aux gros fichiers ?  
  Opérations très coûteuses, taille maximale limitée, lenteur par rapport à AES. 
- Différences entre paramètres publique / privée ?  
  Publique : modulo n, exposant public e. Privée : n, e, exposant privé d, premiers p et q, paramètres dérivés. 
- Rôle du modulo ?  
  C’est le produit de deux grands premiers, l’espace de travail dans lequel se font les opérations RSA . 
- Pourquoi RSA pour chiffrer une clé AES ?  
  RSA est utilisé pour chiffrer **une petite clé** AES, puis AES chiffre le gros fichier : c’est le chiffrement hybride (rapide et sécurisé). 


# D. Signature numérique

## 1. Création et signature

Créer le fichier :

nano contrat.txt

Empreinte (par exemple SHA-256) :

openssl dgst -sha256 contrat.txt

Signer avec ta clé privée :

openssl dgst -sha256 -sign rsa_private.pem -out contrat.sig contrat.txt


## 2. Vérification
Vérifier la signature avec la clé publique :

openssl dgst -sha256 -verify rsa_public.pem -signature contrat.sig contrat.txt


## 3. Réponses rapides
Que se passe-t-il après modification ?
La vérification échoue : l’empreinte ne correspond plus à la signature.

Pourquoi ?
Le hash change complètement dès qu’un bit change, donc la signature ne correspond plus.

Rôle du hachage dans la signature ?
On signe le hash, pas le fichier complet : plus rapide et détecte la moindre modification.

Différence signature numérique / chiffrement ?
Signature : prouver l’authenticité et l’intégrité, via la clé privée pour signer et la publique pour vérifier. Chiffrement : assurer la confidentialité.