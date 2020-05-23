# all-techno
https://github.com/juliencoste/fake-backend.git
https://github.com/juliencoste/cursus-devops.git

projet:

1- Build and deploy (1.1 and 1.2 in 4DOC_TP_v1.0.pdf)
Build the image using Dockerfile and docker-compose.yml to deploy the application
NB: battleboat-gh-pages is here https://github.com/billmei/battleboat.git

2- Automate de deployment 
Use ansible to deploy the build and deploy the application using Docker

3- Automate the Building
Use Jenkins to build and push the app image in your private registry (or dockerhub if you prefer)

etape

1/cree infrastructure (1 vm docker) (1vm ansible) (1vm jenkins) (1vm registre privé)
2/build and deploy api fake-bakend-battleboat en local sur vm docker (recup du code sur github)
3/créé le registre privé
4/créé le role ansible qui 1/build 2/push sur registre privé 3/deploy sur docker en locale
5/deployer sur machine distante avec le playbook
6/faire le job sur jenkins qui build push et deploy en local
7/faire le job sur jenkins sur serveur distant (avec plugin ansible)


1/###cree infrastructure###

sur aws cree dans cloud formation les VM avec les stack qui vont bien

2/###build and deploy### ###sur vm docker###

#instalation des prérequis

sudo yum install git , vim
sudo curl -L "https://github.com/docker/compose/releases/download/1.25.5/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose

#build et deploiement

git clone https://github.com/juliencoste/fake-backend.git
cd fake-backend
docker-compose up -d
on test en se conectant sur l'ip public (/health pour verifier la base de donné)
docker-compose down

###private registy### ###sur vm registry#

#ouvrir le port 5000 sur la security group aws
 
#on crée le dossier de stockage sur notre machine hôte :
mkdir data

#on lance l'image registry:2 avec les ports et un volume persistant
docker run -d \
    -p 5000:5000 \
    --restart=always \
    --name private-registry \
    -e REGISTRY_STORAGE_DELETE_ENABLED=true \
    -v "$(pwd)"/data:/var/lib/registry \
    registry:2

#on recupere l'ip privé du conteneur
docker inspect id-conteneur

#on deploy le front pour le repository
docker run -d \
-p 80:80 \
-e REGISTRY_URL=http://ip-du-conteneur-registry:5000 \
-e DELETE_IMAGES=true \
-e REGISTRY_TITLE="private registry julien" \
joxit/docker-registry-ui:static

#on ajoute le droit http sur le fichier conf
sudo vim /usr/lib/systemd/system/docker.service

#ExecStart=/usr/bin/dockerd (on ajoute dans cette ligne)
--insecure-registry ipprivé:5000
--insecure-registry ippublic:5000

#on restart les service
sudo systemctl daemon-reload
sudo service docker restart

#on restart les conteneur
docker start idconteneur

#on tag et on push une image
docker pull hello-world
docker tag hello-world ipprivé:5000/hello-world
docker push ipprivé:5000/hello-world

#on accede a notre registry avec l'ip public de la vm
#on peut suprimer l'image depuis la page web

###ansible###

gitclone https://github.com/juliencoste/cursus-devops.git

#cree clé ssh pour recuperer le code sur github
ssh-keygen -t rsa
on copie la clé public sur github

#modifier le fichier 
sudo yum install vim
vim cursus-devops/ansible/group_vars/all.yml
#cette ligne
fake-backend_source_repo: "git@github.com:juliencoste/fake-backend.git"
et modifier tout les gitlab en github

#on cree le fichier qui contient les variables avec les mots de passe
vim cursus-devops/ansible/files/secrets/devops.yml
(docker-hub github-cle privé)
#ensuite on l'encrypt
ansible-vault encrypt devops.yml (ansible)


#modifier le role student-list en role fake-backend
voir le cursus-devops/ansible/role/fake-backend/task/main.yml dans github

#créé le playbook qui appel le role fake-backend
voir le cursus-devops/ansible/install-fake-backend.yml

#on lance le playbook en local
cd cursus-devops/ansible
ansible-playbook install-fake-backend.yml --ask-vault-pass


#on test sa conection ssh avec l'hote
on copie sa cle .pem
on effectue un chmod 400 sur celle ci pour changer les droits dessus
ssh -i cursus-devops/ansible/files/secrets/devops.pem centos@ip 


#on cree le fichier ansible.cfg la ou se trouve le playbook pour indiqué la clé ssh le nom d'utilisateur et l'option de ne pas verifier le fingerprint
[defaults]
host_key_checking = False
private_key_file=./files/secrets/devops.pem
remote_user=centos
inventory = ./hosts (fait car il ne trouvais pas le fichier tout seul)

#on modifie le fichier hosts pour rajouter un groupe docker
[docker]
ip-privé-vm-docker

#on test la conection
ansible docker -m ping

#j'encrypt la clé pour pouvoir la push dans mon repo et l'utilisé plus tard
chmod 777 devops.pem
ansible-vault encrypt devops.pem (ansible)

#on modifie le playbook pour l'utilisé sur une machine distante
cp install-fake-backend.yml install-fake-backend-distant.yml
et je modifie le hosts dans le fichier en mettant le groupe docker

#on install docker pour python sur la machine distante via un role ansible (prerequis pour manipulé avec ansible ) sur ansible
ansible-galaxy install -r roles/requirements.yml
vim install_docker.yml (pour modifier le hosts)
ansible-playbook install_docker.yml

#on test le playbook sur docker distant
ansible-playbook install-fake-backend-distant.yml --ask-vault-pass

#je push mon repo pour conservé mes codes
git add .
git commit -m "first commit"
git push origin master

###jenkins### jenkins:user/bitnami

#on ajoute le droit http sur le fichier conf
sudo vim /usr/lib/systemd/system/docker.service

#ExecStart=/usr/bin/dockerd (on ajoute dans cette ligne)
--insecure-registry ip-privé-jenkins:5000
--insecure-registry ip-public-jenkins:5000

#on restart les service
sudo systemctl daemon-reload
sudo service docker restart

#on créé nos credentials (pas besoin pour cette exercice)
pour github
dockerhub

#on créé le job avec un script shell#

#!/bin/bash

set +x
git clone https://github.com/juliencoste/fake-backend.git
cd fake-backend
docker build -t 34.203.209.37:5000/fake-backend:jenkins .
docker push 34.203.209.37:5000/fake-backend:jenkins
rm docker-compose.yml
mv docker-compose8181.yml docker-compose.yml
docker-compose up -d

#cocher la case Delete workspace before build starts

#ouvrir le port 8181 sur le groupe de securité#

#on peut lancer le build

#créé un deuxieme job pour piloter ansible avec jenkins

#!/bin/bash

set +x
git clone https://github.com/juliencoste/cursus-devops.git
echo $password_vault > /tmp/vaultpassword.pem
ansible-vault decrypt $WORKSPACE/cursus-devops/ansible/files/secrets/devops.pem --vault-password-file /tmp/vaultpassword.pem
chmod 400 $WORKSPACE/cursus-devops/ansible/files/secrets/devops.pem
cd $WORKSPACE/cursus-devops/ansible/
ansible-playbook install-fake-backend-distant.yml --vault-password-file /tmp/vaultpassword.pem
rm -rf /tmp/vaultpassword.pem

#je cree un secret-text dans les credential pour le mot de passe vault


#on peut lancer le build


####probleme rencontré:####

repository privé:

comment push sur http
comment suprimé mes images sur le front

ansible:

créé le role
declaration de la clé ssh
fichier hosts qui n'etait pas reconnue

jenkins
faire le job sur machine distante:
declaration de la clé ssh
déclaration du mot de passe vault


####probleme a resoudre#### 
le playbook me cree un fichier mysql sur la vm lors du deploiement (faire un volume avec docker volume dans le playbook)
trouver comment utilisé le clé ssh vaulté sur ansible et jenkins (sans avoir besoin de la decrypté)
trouver une meilleur methode de gerer le mot de passe vault (dans jenkins)
le job pour piloter ansible avec jenkins ne se termine pas du a une version ancienne de ansible (essayé de la mettre a jour)

