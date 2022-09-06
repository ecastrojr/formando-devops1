# Desafio Linux

# Preparação do ambiente

Sugerimos utilizar um sistema unix (linux, macos, \*bsd) ou [WSL](https://docs.microsoft.com/pt-br/windows/wsl/install).

Você precisa instalar o [VirtualBox](https://www.virtualbox.org) e opcionalmente (recomendado) o [Vagrant](https://www.vagrantup.com)

Baixe esse repositório e execute:

```
git clone https://github.com/getupcloud/formando-devops.git
cd desafio-linux
vagrant up
```

Para entrar na VM e iniciar as tarefas, execute:

```
vagrant ssh
```

O endereço IP público da VM pode ser obtido com o comando `ip addr show dev eth1`.

Você pode reiniciar a VM a qualquer momento utilizando a GUI do próprio VirtualBox.

## 1. Kernel e Boot loader

O usuário `vagrant` está sem permissão para executar comandos root usando `sudo`.
Sua tarefa consiste em reativar a permissão no `sudo` para esse usuário.

Dica: lembre-se que você possui acesso "físico" ao host.

**`Resposta:`**

Reiniciado a VM, pressionado "ESC" para mostrar as opções de boot, apertando `e` na primeira opção de boot para editar.
```
Na linha onde tem "root ro no_timer_check...":
Trocar "ro" por "rw init=/sysroot/bin/sh"
```
Apertando `Ctrl+X` para entrar no modo single user

```bash
#trocar o sistema
chroot /sysroot
#trocar a senha do root
passwd
#forçar relabeling para o SELinux
touch /.autorelabel

#sair e reiniciar
exit
logout
reboot
```
`Agora sabemos a senha do root`

Após acessar o ssh novamente com o usuário vagrant, trocar para root e incluir o usuário vagrant como membro do grupo wheels.
```bash
$ su -
Password: 
usermod -aG wheel vagrant 
exit
```
Pronto agora conhecemos a senha do usuário root e usuário vagrant pode utilizar o sudo. `Senha do usuário vagrant é "vagrant"`

## 2. Usuários

### 2.1 Criação de usuários

Crie um usuário com as seguintes características:

- username: `getup` (UID=1111)
- grupos: `getup` (principal, GID=2222) e `bin`
- permissão `sudo` para todos os comandos, sem solicitação de senha

**`Resposta:`**
```bash
#criando o grupo getup com GID 2222
sudo groupadd -g 2222 getup
#criando o usuário getup com UID 1111 e membro dos grupo getup, bin e wheel
sudo useradd -u 1111 -g 2222 -m -G bin,wheel getup
passwd getup
```

```bash
[getup@centos8 ~]$ id
uid=1111(getup) gid=2222(getup) groups=2222(getup),1(bin),10(wheel)
```

Editado arquivo sudoers
```bash
sudo visudo
```
`de:`
```bash
## Allows people in group wheel to run all commands
%wheel  ALL=(ALL)       ALL

## Same thing without a password
# %wheel        ALL=(ALL)       NOPASSWD: ALL
```
`Para:`
```bash
## Allows people in group wheel to run all commands
#%wheel  ALL=(ALL)       ALL

## Same thing without a password
%wheel        ALL=(ALL)       NOPASSWD: ALL
```


## 3. SSH

### 3.1 Autenticação confiável

O servidor SSH está configurado para aceitar autenticação por senha. No entanto esse método é desencorajado
pois apresenta alto nivel de fragilidade. Voce deve desativar autenticação por senhas e permitir apenas o uso
de par de chaves.

**`Resposta:`**

```bash
sudo sed -i 's,PasswordAuthentication yes,PasswordAuthentication no,g' /etc/ssh/sshd_config
sudo systemctl restart sshd
```

### 3.2 Criação de chaves

Crie uma chave SSH do tipo ECDSA (qualquer tamanho) para o usuário `vagrant`. Em seguida, use essa mesma chave
para acessar a VM.

**`Resposta:`**

```bash
cd ~/.ssh
ssh-keygen -t ed25519 -C "chave para desafio linux getup"
ssh-copy-id -i desafiolinux.pub vagrant@192.168.0.26
ssh vagrant@192.168.0.26
```

### 3.3 Análise de logs e configurações ssh


Utilizando a chave do arquivos `id_rsa-desafio-devel.gz.b64` deste repositório, acesso a VM com o usuário `devel`

**`Resposta:`**



```bash
#decodificando 
base64 --decode id_rsa-desafio-linux-devel.gz.b64 > id_rsa-desafio-linux-devel.gz
#descompactando
gzip -d id_rsa-desafio-linux-devel.gz
chmod 400 id_rsa-desafio-linux-devel
ssh -i id_rsa-desafio-linux-devel devel@192.168.0.28 -v
#verificado problema no arquivo
Load key "id_rsa-desafio-linux-devel": error in libcrypto
cat -e id_rsa-desafio-linux-devel
#convertendo arquivo para o formato adequado
dos2unix id_rsa-desafio-linux-devel
ssh -i id_rsa-desafio-linux-devel devel@192.168.0.28 -vvv
[vagrant@centos8 ~]$ sudo journalctl -u sshd
#encontrado erro
Authentication refused: bad ownership or modes for file /home/devel/.ssh/authorized_keys
#Permissões do arquivo estava com 777:
-rwxrwxrwx. 1 devel devel  617 Sep  3 22:54 authorized_keys
chmod 600 authorized_keys
-rw-------. 1 devel devel  617 Sep  3 22:54 authorized_keys
#nova tentativa
ssh -i id_rsa-desafio-linux-devel devel@192.168.0.28
[devel@centos8 ~]$ 
```


## 4. Systemd

Identifique e corrija os erros na inicialização do serviço `nginx`.
Em seguida, execute o comando abaixo (exatamente como está) e apresente o resultado.
Note que o comando não deve falhar.

```
curl http://127.0.0.1
```
**`Resposta:`**

Não está respondendo! O serviço do nginx parado

```bash
curl http://127.0.0.1
curl: (7) Failed to connect to 127.0.0.1 port 80: Connection refused
sudo systemctl status nginx.service 
sudo systemctl start nginx.service 
sudo journalctl -u nginx
  # invalid number of arguments in "root" directive in /etc/nginx/nginx.conf:45
sudo nginx -t
  # invalid number of arguments in "root" directive in /etc/nginx/nginx.conf:45
sudo vim /etc/nginx/nginx.conf
```
`";" faltando no final de uma das linhas`

`NGINX estava configurado para responder na porta 90 e não na 80`


Alterado as linhas do arquivo /etc/nginx/nginx.conf
```vim
de:
linha 39:        listen       90 default_server;
Linha 40:        listen       [::]:90 default_server;
linha 42:        root         /usr/share/nginx/html

para:
linha 39:        listen       80 default_server;
Linha 40:        listen       [::]:80 default_server;
linha 42:        root         /usr/share/nginx/html;

```
```bash
sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```
Alterado porta e adicionado ; ao final da linha o comando `nginx -t` não acusava mais erro.
Porém mesmo após as modificações o serviço não iniciou, nos logs estava claro o problema.
```
Process: 2600 ExecStart=/usr/sbin/nginx -BROKEN (code=exited, status=1/FAILURE)
```
Editado o arquivo `/usr/lib/systemd/system/nginx.service` 

```bash
#removendo o parametro "-BROKEN"
sudo vim /usr/lib/systemd/system/nginx.service
sudo systemctl daemon-reload
sudo systemctl start nginx.service 
#Serviço iniciou com sucesso.
curl http://127.0.0.1
Duas palavrinhas pra você: para, béns!
```



## 5. SSL

### 5.1 Criação de certificados

Utilizando o comando de sua preferencia (openssl, cfssl, etc...) crie uma autoridade certificadora (CA) para o hostname `desafio.local`.
Em seguida, utilizando esse CA para assinar, crie um certificado de web server para o hostname `www.desafio.local`.

**`Resposta:`**

Gerando a chave privada e o certificado raiz
```bash
mkdir ~/certs
cd certs/
openssl genrsa -des3 -out desafio.local.key 2048
openssl req -x509 -new -nodes -key desafio.local.key -sha256 -days 1825 -out desafio.local.pem
#Instalando o certificado raiz no centOS
sudo cp desafio.local.pem /etc/pki/ca-trust/source/anchors/
sudo update-ca-trust 
```
Criando o certificado assinado pela CA

```bash
#Criando a chave privada
openssl genrsa -out www.desafio.local.key 2048
#Criando a solicitação de assinatura de certificado (CSR)
openssl req -new -key www.desafio.local.key -out www.desafio.local.csr
#Criando arquivo para definir SAN
vim www.desafio.local.ext
```
Conteúdo do arquivo `www.desafio.local`
```vim
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, nonRepudiation, keyEncipherment, dataEncipherment
subjectAltName = @alt_names

[alt_names]
DNS.1 = desafio.local
DNS.2 = www.desafio.local
```
Criando o certificado, utilizando o CSR, a chave privada da CA, o certificado da CA e o arquivo SAN:
```bash
openssl x509 -req -in www.desafio.local.csr -CA desafio.local.pem -CAkey desafio.local.key \
-CAcreateserial -out www.desafio.local.crt -days 825 -sha256 -extfile www.desafio.local.ext 
```


### 5.2 Uso de certificados

Utilizando os certificados criados anteriormente, instale-os no serviço `nginx` de forma que este responda na porta `443` para o endereço
`www.desafio.local`. Certifique-se que o comando abaixo executa com sucesso e responde o mesmo que o desafio `4`. Voce pode inserir flags no comando
abaixo para utilizar seu CA.

```
curl https://www.desafio.local
```
**`Resposta:`**

Modificando arquivo `nginx.conf` para habilitar
```bash
sudo vim /etc/nginx/nginx.conf
```
```vim
#Settings for a TLS enabled server.

    server {
        listen       443 ssl http2 default_server;
        listen       [::]:443 ssl http2 default_server;
        server_name  _;
        root         /usr/share/nginx/html;

        ssl_certificate "/etc/pki/nginx/www.desafio.local.crt";
        ssl_certificate_key "/etc/pki/nginx/private/www.desafio.local.key";
        ssl_session_cache shared:SSL:1m;
        ssl_session_timeout  10m;
        ssl_ciphers PROFILE=SYSTEM;
        ssl_prefer_server_ciphers on;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }
    }

}

```

```bash
#criando as pastas e copiando os arquivos
sudo mkdir /etc/pki/nginx
sudo cp www.desafio.local.crt /etc/pki/nginx/www.desafio.local.crt
sudo mkdir /etc/pki/nginx/private
sudo cp www.desafio.local.key /etc/pki/nginx/private/www.desafio.local.key
#reiniciado o serviço
sudo systemctl restart nginx.service 
ss -atunp
#Definindo a resolução de nome para www.desafio.local e desafio.local
sudo vim /etc/hosts 
```
```vim
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4 www.desafio.local desafio.local
```
```bash
#testando
[vagrant@centos8 certs]$ curl http://www.desafio.local
Duas palavrinhas pra você: para, béns!
[vagrant@centos8 certs]$ curl https://www.desafio.local
Duas palavrinhas pra você: para, béns!
[vagrant@centos8 certs]$ curl https://desafio.local
Duas palavrinhas pra você: para, béns!

```


## 6. Rede

### 6.1 Firewall

Faço o comando abaixo funcionar:

```
ping 8.8.8.8
```
**`Resposta:`**

Comando já estava funcionando!
```bash
[vagrant@centos8 system]$ ping -c 3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=29.8 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=31.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=63 time=34.6 ms

--- 8.8.8.8 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2004ms
rtt min/avg/max/mdev = 29.798/32.046/34.551/1.959 ms

```


### 6.2 HTTP

Apresente a resposta completa, com headers, da URL `https://httpbin.org/response-headers?hello=world`


**`Resposta:`**
```bash
[vagrant@centos8 system]$ curl -i https://httpbin.org/response-headers?hello=world
HTTP/2 200 
date: Sun, 04 Sep 2022 00:56:36 GMT
content-type: application/json
content-length: 89
server: gunicorn/19.9.0
hello: world
access-control-allow-origin: *
access-control-allow-credentials: true

{
  "Content-Length": "89", 
  "Content-Type": "application/json", 
  "hello": "world"
}
```

## 7. Logs

Configure o `logrotate` para rotacionar arquivos do diretório `/var/log/nginx`

**`Resposta:`**

Criado arquivo '/etc/logrotate.d/nginx' com o conteúdo:

```bash
[vagrant@centos8 ~]$ sudo vim /etc/logrotate.d/nginx
```
```vim
/var/log/nginx/*log {
    daily
    rotate 10
    missingok
    notifempty
    compress
    sharedscripts
    postrotate
    /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```
Forçar a execução do logrotate e confirmando execução:
```bash
[vagrant@centos8 ~]$ sudo logrotate -f /etc/logrotate.d/nginx
[vagrant@centos8 ~]$ cat /var/lib/logrotate/logrotate.status 
logrotate state -- version 2
"/var/log/nginx/access.log" 2022-9-4-3:46:2
[vagrant@centos8 ~]$ sudo ls -lha /var/log/nginx/
total 12K
drwxrwx---. 2 nginx root   86 Sep  4 03:46 .
drwxr-xr-x. 9 root  root 4.0K Sep  4 03:27 ..
-rw-rw-r--  1 nginx root    0 Sep  4 03:46 access.log
-rw-r--r--  1 root  root  163 Sep  4 02:47 access.log.1.gz
-rw-rw-r--  1 nginx root    0 Sep  4 03:46 error.log
-rw-r--r--  1 root  root  236 Sep  4 02:32 error.log.1.gz

```


## 8. Filesystem

### 8.1 Expandir partição LVM

Aumente a partição LVM `sdb1` para `5Gi` e expanda o filesystem para o tamanho máximo.

**`Resposta:`**
```bash
sudo fdisk -l
sudo parted /dev/sdb print free
sudo parted /dev/sdb resizepart 1 5368MB
#  1      1049kB  5368MB  5367MB  primary               lvm
sudo pvresize /dev/sdb1
#  /dev/sdb1  data_vg    lvm2 a--    <5.00g 4.00g
#  data_vg      1   1   0 wz--n-   <5.00g 4.00g
sudo lvextend -l +100%FREE /dev/data_vg/data_lv
#  data_lv data_vg    -wi-ao----   <5.00g                           
sudo resize2fs /dev/data_vg/data_lv
# linha do resultado comando df -hT
/dev/mapper/data_vg-data_lv ext4      5.0G  4.0M  4.7G   1% /data
```
### 8.2 Criar partição LVM

Crie uma partição LVM `sdb2` com `5Gi` e formate com o filesystem `ext4`.

**`Resposta:`**
```bash
sudo cfdisk /dev/sdb
sudo partprobe /dev/sdb
sudo pvdisplay
sudo pvcreate /dev/sdb2
sudo vgcreate data2_vg /dev/sdb2
sudo lvcreate -n data2_lv -l+100%FREE data2_vg
sudo mkfs.ext4 /dev/data2_vg/data2_lv
sudo mount /dev/data2_vg/data2_lv /data2
# linha do resultado comando df -hT
/dev/mapper/data2_vg-data2_lv ext4      4.9G   20M  4.6G   1% /data2
```

### 8.3 Criar partição XFS

Utilizando o disco `sdc` em sua todalidade (sem particionamento), formate com o filesystem `xfs`.

**`Resposta:`**
```bash
sudo cfdisk /dev/sdc
sudo dnf install xfsprogs
sudo mkfs.xfs /dev/sdc
sudo parted /dev/sdc print free
Model: ATA VBOX HARDDISK (scsi)
Disk /dev/sdc: 10.7GB
Sector size (logical/physical): 512B/512B
Partition Table: loop
Disk Flags: 
Number  Start  End     Size    File system  Flags
 1      0.00B  10.7GB  10.7GB  xfs
# linha do resultado comando df -hT
/dev/sdc                      xfs        10G  423M  9.6G   5% /xfs
```
