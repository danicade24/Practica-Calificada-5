# Practica Calificada 5

**Nombre:** Daniela Cadena Villanueva

**email:** daniela.cadena.v@uni.pe

-----
## Ejercicio 1: Configuración básica del sistema

En el ejecicio 1 debemos de:
- Configurar la maquina virtual y agregar las tareas de crear usuarios y grupos
- Actualizar los paquetes del SO
- Crear un usuario y asignarlo a un grupo

Primero configuramos nuestro archivo Vagrantfile:

1. Crear el archivo dentro de `vagrant/` ejecutamos:

```bash
vagrant init ubuntu/focal64
``` 
2. configuramos nuestro archivo `Vagrantfile`
![](docs/ej1/ej1vagrantfile.png)

3. Configuramos nuestro playbook `site.yml`

```yaml
---
- name: Aprovisionador de ansible para configurar la VM
  hosts: all
  become: yes
  become_method: sudo
  remote_user: vagrant
  tasks:
    - import_tasks: ansible/ejercicio1/main.yml
```

4. Configuramos las tareas en `ansible/ejercicio1/main.yml`

![](docs/ej1/tasks.png)

![](docs/ej1/terminal.png)

5. En VirtualBox observamos que la máquina esta corriendo

![](docs/ej1/ej1vbox.png)

6. Nos conectamos a nuestra máquina remota mediante ssh con `vagrant ssh` y comprobamos que se creo el usuario devuser con `id devuser`

![](docs/ej1/ej1_ssh.png)

También comprobamos que están presentes las demás configuraciones que se realizaron con las tareas

![](docs/ej1/tareas_terminal.png)
----

## Ejercicio 2: Implementación de servicion web con seguridad básica

1. Actualizamos nuestro playbook.yml `site.yml` según los requerimientos del ejercicio:
```yaml
---
- name: Aprovisionador de ansible para configurar la VM
  hosts: all
  become: yes
  become_method: sudo
  remote_user: vagrant
  tasks:
    - import_tasks: ansible/ejercicio1/main.yml
    - import_tasks: ansible/ejercicio2/main.yml
  handlers:
    - include_tasks: handlers/main.yml
```

2. Creamos las tareas:
 - Instalar Nginx
 - Generar certificados SSL autofirmados
 - Configuración de Nginx para utilizarlo con SSL
 - Habilita y configurar UFW para permitir tráfico en los puertos necesarios

```yaml
---
- name: Instalar nginx
  apt: 
    name: nginx
    state: latest
    update_cache: yes

- name: Instalar OpenSSL
  apt:
    name: openssl
    state: present
    update_cache: yes

- name: Crear directorio para certificados SSL
  file:
    path: /etc/ssl/certs
    state: directory
    mode: '0755'

- name: Generar clave privada
  community.crypto.openssl_privatekey:
    path: /etc/ssl/private/mi_certificado.key
    size: 2048

- name: Generar certificado SSL autofirmado
  community.crypto.x509_certificate:
    path: /etc/ssl/certs/mi_certificado.crt
    privatekey_path: /etc/ssl/private/mi_certificado.key
    provider: selfsigned

- name: Configurar Nginx para usar con SSL
  ansible.builtin.template:
    src: templates/nginx_ssl.conf.j2
    dest: /etc/nginx/sites-available/default
    mode: '0644'
  notify:
    - Reiniciar Nginx

- name: Habilitar y configurar el firewall UFW 
  ufw:
    state: enabled
    policy: allow

- name: Permitir tráfico en los puertos necesarios
  ufw:
    rule: allow
    port: "22,80,443" # SSH, HTTP, HTTPS
    proto: tcp
```

4. Creamos el manejador en `handlers/main.yml` 
```yaml
---
- name: Reiniciar Nginx
  ansible.builtin.service:
    name: nginx
    state: restarted
```

5. Creamos la platilla que nos permite configurar Nginx para usar los certificados SSL y servidores HTTPS

**Bloque para el Servidor HTTPS**

```nginx
server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name localhost;

    ssl_certificate /etc/nginx/ssl/nginx-selfsigned.crt;
    ssl_certificate_key /etc/nginx/ssl/nginx-selfsigned.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        root /var/www/html;
        index index.html;
    }
}
```

El primer bloque nos permite configurar el puerto 443 para el servidor HTTPS también se especifican las rutas donde se esncuentra el certificado SSL (`/etc/nginx/ssl/nginx-selfsigned.crt`) y la clave privada (`/etc/nginx/ssl/nginx-selfsigned.key`).

La línea `ssl_protocols TLSv1.2 TLSv1.3` define los protocolos TLS que son permitidos y la línea `ssl_ciphers HIGH:!aNULL:!MD5` define las cifras criptográficas que permiten las conexiones seguras. 


**Bloque para el Servidor HTTP**

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name localhost;
    return 301 https://$host$request_uri;
}
```

El servido va a escuchar en el puerto 80, definimos el nombre del servidor como `localhost` por úiltimo `return 301 https://$host$request_uri` se encarga de redirigir las solicitudes HTTP a la versón de HTTPS.


6. Actualizamos nuestra VM con las nuevas configuraciones

![](docs/ej2/tasks.png)

4. Ingresamos a nuestra vm mediante `vagrant ssh` y comprobamos el estado de nuestras tareas:

![](docs/ej2/instalar_nginx.png)

![](docs/ej2/ssl_ceritificado.png)

![](docs/ej2/ufw.png)

![](docs/ej2/localhost.png)

---
 **Nota**
- Para la configuración de generación de certificados autofirmados consulté en [la siguiente página](https://docs.ansible.com/ansible/latest/collections/community/crypto/docsite/guide_selfsigned.html)
- Revisé [la siguiente pagina](https://docs.ansible.com/ansible/latest/playbook_guide/playbooks_handlers.html) para configurar los handlers

----

## Ejercicio 3: Despliegue de una palicación web con balanceador de carga


[Crear directorio](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/file_module.html#ansible-collections-ansible-builtin-file-module)

[Template](https://docs.ansible.com/ansible/latest/collections/ansible/builtin/template_module.html)

[Configuracion del directorio](https://www.linode.com/docs/guides/how-to-enable-disable-website/)

[Systemd](https://docs.ansible.com/ansible/5/collections/ansible/builtin/systemd_module.html)

---

