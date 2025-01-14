## Implémentation de SSL dans Nginx

├── ansible.cfg

├── files

│   ├── nginx.crt

│   └── nginx.key

├── inventory

│   └── vagrant.ini

├── README.md

├── templates

│   ├── index.html.j2

│   └── nginx.conf.j2

├── Vagrantfile

└── webservers-tls.yml

**Génération d'un certificat auto-signé**

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
-subj /CN=localhost \
-keyout files/nginx.key -out files/nginx.crt
```

**Création du playbook (webservers-tls.yml)**

```yaml
---

- name: Installation et configuration de NGINX
  hosts: webservers
  become: true

  vars:
    tls_dir: /etc/nginx/ssl/
    key_file: nginx.key
    cert_file: nginx.crt
    conf_file: /etc/nginx/sites-available/default
    server_name: localhost

  handlers:
    - name: Restart nginx
      service:
       name: nginx
       state: restarted

  tasks:
    - name: Installation de NGINX
      package: 
        name: nginx
        update_cache: yes

    - name: Création du répertoire pour les certificats TLS
      file:
        path: "{{ tls_dir }}"
        state: directory
        mode: '0750'
      notify: Restart nginx

    - name: Copie des fichiers TLS
      copy:
        src: "{{ item }}"
        dest: "{{ tls_dir }}"
        mode: '0600'
      loop:
        - "{{ key_file }}"
        - "{{ cert_file }}"
      notify: Restart nginx


    - name: Copie des fichiers de configuration NGINX
      template:
        src: nginx.conf.j2
        dest: "{{ conf_file }}"
        mode: '0644'
      notify: Restart nginx

    - name: Activer la configuration
      file: >
        dest=/etc/nginx/sites-enabled/default
        src=/etc/nginx/sites-available/default
        state=link

    - name: Copie de la page web index.html
      template: >
        src=index.html.j2
        dest=/usr/share/nginx/html/index.html

    - name: Restart nginx
      meta: flush_handlers

    - name: "Test it! https://localhost:8443/index.html"
      delegate_to: localhost
      become: false
      uri:
        url: 'https://localhost:8443/index.html'
        validate_certs: false
        return_content: true
      register: this
      failed_when: this.status != 200
      tags:
        - test


...
```

**Création du fichier template J2 de la configuration du site nginx (nginx.conf.j2)**

```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server ipv6only=on;
        
        listen 443 ssl;
        ssl_protocols TLSv1.2;
        ssl_prefer_server_ciphers on;
        root /usr/share/nginx/html;
        index index.html;
        server_tokens off;
        add_header X-Frame-Options DENY;
        add_header X-Content-Type-Options nosniff;
        
        server_name {{ server_name }};
        ssl_certificate {{ tls_dir }}{{ cert_file }};
        ssl_certificate_key {{ tls_dir }}{{ key_file }};
        
        location / {
            try_files $uri $uri/ =404;
        }
}
```