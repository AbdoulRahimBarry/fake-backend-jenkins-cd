********************Jenkins_DC******************************************
------------------Remqrque-----------------------------------
sudo ln -s /usr/local/bin/docker-compose /usr/bin/ ( )
	/usr/local/bin/docker-compose : lien existant
	/usr/bin/ : lien symbolique
        lorsque j'execute mes comande docker, le demon docker chercher les commande dans /usr/bin,reseol on cree le lien entre les deux dossier
docker image -a -q (donne les id de tout le image)


1- Creaction des credential
    Definir les identifiant pour ce connecté soit sur docker hub, git ou une connexion sur une machine distant	

2- Gestion de code source ---------------
 mettre le lien du repository : https://github.com/AbdoulRahimBarry/jenkins.git

3- Environnement de build:
  Building choisir Username and password(separated)
	* User secret text or file

4-  Build:
    Executé un script shell

	#!/bin/bash
	set +x
	cd fake-backend
	pwd
	docker build -t barry2abdulrahim/game_image:version1 .
	docker login --username=$docker_login --password=$docker_password
	docker push barry2abdulrahim/game_image:version1
	cd ..
	cd role
	ansible-galaxy install -r requirements.yml

    Cliniqué une etape de build
	Choisir > Invoke Ansible Playbook
	Playbook path : schemain du playbook,  coché >File or host list: schemain du fichier host
    Vault Credentials : importer le fichier qui contient le login des fichier vaulté ('centos')
    Credentials: > Ajouter > choisir text afin de coler la clé .pem pour se connecté a la machine distant

