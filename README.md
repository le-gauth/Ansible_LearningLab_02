# Ansible_LearningLab_02

## Création de la VM

**Vagrant file**

```ruby
# -*- mode: ruby -*-
# vi: set ft=ruby :

Vagrant.configure("2") do |config|
	config.vm.box = "ubuntu/focal64"
	config.vm.hostname = "testserver"
	config.vm.network "forwarded_port", id: 'ssh', guest: 22, host: 2202, host_ip: "127.0.0.1", auto_correct: false
	config.vm.network "forwarded_port", id: 'http', guest: 80, host: 8080, host_ip: "127.0.0.1"
	config.vm.network "forwarded_port", id: 'https', guest: 443, host: 8443, host_ip: "127.0.0.1"

	if Vagrant.has_plugin?("vagrant-vbguest")
		config.vbguest.auto_update = false
	end

	config.vm.provider "virtualbox" do |virtualbox|
		virtualbox.name = "ch03"
	end

end
```

## Développement du premier playbook et fichiers associés

>Ce premier playbook permet d'installer et de configurer NGINX avec une simple page web en HTTP sur la VM

> **<u>Arborescence finale :</u>**
./Ansible_LearningLab_02
├── ansible.cfg
├── files
│   └── nginx.conf
├── inventory
│   └── vagrant.ini
├── README.md
├── templates
│   └── index.html.j2
├── Vagrantfile
└── webservers.yml

**Playbook webservers.yml**

```yaml
---

- name: Installation et configuration de NGINX
  hosts: webservers
  become: true
  tasks:
    - name: Installation de NGINX
	  package: 
	    name: nginx
	    update_cache: yes

	- name: Copie des fichiers de configuration NGINX
	  copy:
	    src: files/nginx.conf
		dest: /etc/nginx/sites-available/default

	- name: Activer la configuration
	  file: >
		dest=/etc/nginx/sites-enabled/default
		src=/etc/nginx/sites-available/default
		state=link

	- name: Copie de la page web index.html
	  template: >
		src=templates/index.html.j2
		dest=/usr/share/nginx/html/index.html

	- name: Redémarrage de NGINX pour appliquer la configuration
	  service: name=nginx state=restarted
...
```

**Configuration du site NGINX (nginx.conf)**

```nginx
server {
		listen 80 default_server;
		listen [::]:80 default_server ipv6only=on;

		root /usr/share/nginx/html;
		index index.html index.htm;
		charset utf-8;

		server_name localhost;

		location / {
			try_files $uri $uri/ =404;
		}
}
```

**Page web (index.html.j2)**

```html
<html>
    <head>
        <meta charset="UTF-8">
        <title>Welcome to Ansible</title>
    </head>
    
    <body>
        <h1>Nginx, configuré par Ansible !</h1>
        <p>Le playbook c'est bien exécuté ! BADABOUM !</p>
        <p>Et ça tourne sur la VM {{ inventory_hostname }} !</p>
    </body>
</html>
```

**Inventaire (vagrant.ini)**

```ini
[webservers]
testserver ansible_port=2202

[webservers:vars]
ansible_user = vagrant
ansible_host = 127.0.0.1
ansible_private_key_file = .vagrant/machines/default/virtualbox/private_key
```


## Syntaxe YAML

> L'indentation est très importante (2 espaces)

**Début du fichier**

```yaml
---
```

**Fin du fichier**

```yaml
...
```

**Commentaire**

```yaml
# Ceci est un commentaire, PROUT !
```

**Chaine de caractères**

```yaml
tel quel, pas besoin de guillemets
```

**Chaine de caractères multilignes**

```yaml
message: |+
  Bonjour à toi,

  Que cette nouvelle année soit rempli de....
  PROUT !
  Bien à toi.
  Toto
destinataire: prout nation CEO
```

**Booléen**

```yaml
# Condition vrai
true, True, TRUE, yes, Yes, YES, on, On, ON

# Condition faux
false, False, FALSE, no, No, NO, off, Off, OFF
```

> Toutes ces différentes formes peuvent être utilisées dans un playbook sans souci d'uniformité

**Liste**

```yaml
liste1:
  - chariot
  - prout
  - olala
```

**Dictionnaire**

```yaml
mondictionnaire:
 nom: martin
 prenom: martin
 ville: martinville
 attaque_special: prout atomique
```



## Structure d'un playbook

**Play**

```yaml
- name: Configure webserver with nginx # Nom du playbook
  hosts: webservers # Host ou groupe sur lequel il s'appliquera
  become: true # Permet d'exécuter le playbook avec l'élévation sudo sur la vm, peut spécifier sur les tasks
```

**Task**

```yaml
- name: Ensure nginx is installed # Nom de la tâche
  package: # Module Ansible
	name: nginx # Nom du package
	update_cache: true # apt update
```

>  Le module package vérifie si le paquet spécifier est installé, si il ne l'est pas, le module l'installera.
>  L'option update_cache permet d'effectuer une mise à jour de la liste des paquets disponibles depuis les dépots configuré sur la VM (apt update)