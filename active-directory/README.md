Para criar uma instância do Active Directory usando Vagrant ou Docker com suporte a LDAPS para testes locais, você pode seguir as etapas abaixo. Vamos usar o Docker, pois é uma abordagem mais leve e rápida para este tipo de teste.

Requisitos:

    •	Docker instalado na sua máquina.
    •	Docker Compose instalado (opcional, mas facilita a configuração).
    •	OpenSSL para gerar certificados.

Passo 1: Preparar o ambiente com Docker

    1.	Instalar Docker: Caso você ainda não tenha o Docker instalado, siga as instruções de instalação para o seu sistema operacional no site oficial do Docker.
    2.	Instalar Docker Compose: Se você deseja usar Docker Compose, pode instalar seguindo as instruções de instalação do Docker Compose.

Passo 2: Criar o Dockerfile do Active Directory

    1.	Crie um diretório de trabalho para o seu projeto e entre nele:

mkdir ad-ldaps && cd ad-ldaps

    2.	Dentro desse diretório, crie um arquivo chamado Dockerfile:

touch Dockerfile

    3.	No Dockerfile, adicione o seguinte conteúdo. Isso vai configurar um Active Directory usando o Samba (que suporta AD) e adicionar o suporte a LDAPS:

FROM ubuntu:20.04

ENV DEBIAN_FRONTEND=noninteractive
RUN apt-get update && apt-get install -y samba krb5-user smbclient openssl

# Criar os diretórios necessários

RUN mkdir -p /etc/samba /var/lib/samba/private

# Configurar o samba

COPY smb.conf /etc/samba/smb.conf

# Expor as portas necessárias

EXPOSE 389 636

CMD ["samba", "--no-process-group", "--foreground", "--log-stdout"]

Passo 3: Configurar o Samba (Active Directory)

    1.	No mesmo diretório, crie um arquivo chamado smb.conf com a seguinte configuração básica para o Samba atuar como um controlador de domínio do Active Directory:

[global]
workgroup = EXEMPLO
realm = EXEMPLO.LOCAL
netbios name = DC1
server role = active directory domain controller
idmap_ldb:use rfc2307 = yes
tls enabled = yes
tls keyfile = /etc/samba/tls/privatekey.pem
tls certfile = /etc/samba/tls/certificate.pem
tls cafile = /etc/samba/tls/cacert.pem

[netlogon]
path = /var/lib/samba/sysvol/exemplo.local/scripts
read only = no

[sysvol]
path = /var/lib/samba/sysvol
read only = no

    2.	No diretório do projeto, crie um subdiretório para armazenar os certificados TLS:

mkdir -p tls

Passo 4: Gerar os Certificados TLS

    1.	Gere um certificado autoassinado usando OpenSSL:

openssl req -new -newkey rsa:4096 -days 365 -nodes -x509 \
 -keyout tls/privatekey.pem -out tls/certificate.pem \
 -subj "/C=BR/ST=PR/L=Curitiba/O=Exemplo/OU=TI/CN=exemplo.local"

Isso cria os arquivos privatekey.pem e certificate.pem dentro do diretório tls.

    2.	Mova os arquivos de certificado para o local apropriado no contêiner Samba:

mv tls/privatekey.pem tls/certificate.pem /etc/samba/tls/

Passo 5: Usar Docker Compose (opcional)

Crie um arquivo docker-compose.yml para simplificar o comando de inicialização:

version: '3'
services:
samba-ad:
build: .
ports: - "389:389" - "636:636"
volumes: - ./tls:/etc/samba/tls
environment: - SAMBA_DOMAIN=EXEMPLO - SAMBA_REALM=EXEMPLO.LOCAL

Passo 6: Subir o Contêiner

    1.	Se estiver usando Docker Compose, você pode subir o ambiente com:

docker-compose up

    2.	Caso não use Docker Compose, execute o contêiner diretamente com Docker:

docker build -t samba-ad .
docker run -d -p 389:389 -p 636:636 -v $(pwd)/tls:/etc/samba/tls samba-ad

Isso iniciará um controlador de domínio Active Directory com suporte a LDAPS nas portas 389 (LDAP) e 636 (LDAPS).

Passo 7: Testar a Conexão LDAPS

Agora você pode testar sua conexão com o Active Directory usando LDAPS na aplicação PHP. Certifique-se de que sua aplicação PHP está configurada para se conectar usando a porta 636.

Exemplos de Configuração PHP:

$ldapconn = ldap_connect("ldaps://localhost:636");

if ($ldapconn) {
    ldap_set_option($ldapconn, LDAP_OPT_PROTOCOL_VERSION, 3);
ldap_set_option($ldapconn, LDAP_OPT_REFERRALS, 0);

    $ldapbind = ldap_bind($ldapconn, "CN=Administrator,CN=Users,DC=exemplo,DC=local", "senha");

    if ($ldapbind) {
        echo "Conexão estabelecida e autenticação LDAP com sucesso!";
    } else {
        echo "Erro ao autenticar no LDAP.";
    }

} else {
echo "Erro ao conectar ao servidor LDAP.";
}

Passo 8: Ajustes e Debug

Se você encontrar algum problema, pode verificar os logs do Samba executando:

docker logs samba-ad

Isso exibirá quaisquer erros que possam estar ocorrendo com a configuração do Samba ou as tentativas de conexão via LDAPS.

Com isso, você terá um ambiente local Active Directory com suporte a LDAPS pronto para testar sua aplicação PHP.
