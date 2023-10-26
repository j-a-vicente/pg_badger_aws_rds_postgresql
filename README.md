# Criando uma imagem no Docker com o apache, pg_badger analisando os logs do AWS RDS PostgreSQL.
Neste tutorial vamos instalar o pg_badger, Apache e o AWSClin para baixar os logs do PostgreSQL em uma imagem do Ubuntu no Docker.
Todos os comando serão executado no Windows 10.

## Resumo do container:
O container  será criado com a porta 8585 do computador redirecionada para a porta 80 do container, com este redirecionamento o relatório poderá ser acessado via browser no computador através  da url ```http://localhost:8585```
Para conservar os logs será criado um volume local em outra unidade de disco na máquina.
O redirecionamento da porta 8989 para porta 22 ssh do container  só deverá ser feita caso você queira acessa o container  via ssh.

### Fluxo do projeto.
01 - Instalar o docker (Como já existe muitos artigos sobre este tópico não vou explicar está etapa.).
02 - Baixar a imagem do Ubuntu e criar o container ..
03 - Instalação dos pré-requisitos no linux.
04 - Instalar o ssh (Opcional caso não queira acessar a usa imagem via ssh pode pular esta etapa.).
05 - Instalar e configura o pg_badger.
06 - Configura o RDS PostgreSQL para gera os logs que serão consumido pelo pg_badger.
07 - Instalar e configurando o AWSClin para executar o download dos logs.
08 - Baixando os logs para o container .
09 - Instalando e configurando o Apache.
10 - Movendo o index.html para Apache.
11 - Acessando pg_badger na estão de trabalho.

#### 01 - Instalar o docker (Como já existe muitos artigos sobre este tópico não vou explicar está etapa.).
[Instale o Docker Desktop no Windows](https://docs.docker.com/desktop/install/windows-install/)

#### 02 - Baixar a imagem do Ubuntu e criar o container .
##### Docker
+ Baixar a imagem do ubuntu: ```docker pull ubuntu```.

+ Criar a imagem com os parâmetros.
    + Direcionando os logs para um volume no disco E:\pgbadger\volume na estação para o local dentro do container  /pgbadger/log do contener ```-v "C:\Install\Docker\pgbadger:/pgbadger/log" ```.
    + Redirecionando a porta 8585 da estação para porta 80 do contener para acessar o Apache ```-p 8585:80 ```.
    + Redirecionando a porta 8989 da estação para porta 22 do contener para acessar o acesso ssh ```-p 8989:22 ```.
    + Nome do container  ```-name pgbadger ```.
+ Criando o container  com os parâmetros acima: ```docker run -p 8585:80 -p 8989:22 -v "C:\Install\Docker\pgbadger:/pgbadger/log" --name pgbadger -it ubuntu```.
+ Iniciar o contener: ```docker start pgbadger```.

#### 03 - Instalação dos pré-requisitos no linux.
+ Entra no terminal do contener ```docker exec -it pgbadger /bin/bash```
+ Instalando os pre-requisitos: <p/> 
```
apt-get update -y 
apt-get install iproute2 -y 
apt-get install wget -y 
apt-get install python2 -y 
apt-get install vim -y 
apt-get install htop -y 
apt-get install git -y
apt-get install perl -y  
apt-get install make -y 
apt-get install sudo -y
apt-get install systemctl -y
apt-get install jq -y
```

#### 04 - Instalar o ssh (Opcional caso vc queira acessar a usa imagem via ssh pode pular esta etapa.).
+ Instalar ssh ```apt-get install openssh-server -y```
+ Iniciar o serviço do ssh ```service ssh start```
+ Configura o acesso.
    + Editando o arquivo de regra de acesso  ```vim /etc/ssh/sshd_config ```
    + Acrescente  as linhas no arquivo. 
    ``` 
        port 22
        PermitRootLogin without-password
        PermitRootLogin yes
    ```    
+ Reinicie o serviço do ssh ```service ssh restart```
+ Criar o usuário ```adduser ubuntu```
+ Acessando o container  via ssh ```ssh -p8989 ubuntu@localhost```.
+ Trocando do usuário ubuntu para o usuário root ```su root ```.

#### 05 - Instalar e configura o pg_badger.
```
cd ~/
mkdir git
cd git 
wget https://github.com/darold/pgbadger/archive/refs/tags/v11.8.tar.gz 
tar xzvf v11.8.tar.gz 
cd pgbadger-11.8 
perl Makefile.PL 
make && sudo make install 
pgbadger -V
```

#### 06 - Configura o RDS PostgreSQL para gera os logs que serão consumido pelo pg_badger.



#### 07 - Instalar e configurando o AWSClin para executar o download dos logs.
+ Atualizando linux ```apt-get update -y ``` 
+ instalando o AWSClin ```apt-get install awscli -y ``` escolha a região 2 fuso 135.
+ Configurando  o AWSClin.
    + Acesse a console da AWS, no canto direito na credencial com seu login clique em <b>Credencial de segurança</b>
![image](https://github.com/j-a-vicente/powershell_fileserver_powerbi/blob/main/imagens/Credenciais%20de%20seguran%C3%A7a%20_%20IAM.png?raw=true)</p>

    + Criar chave de acesso.
![image](https://github.com/j-a-vicente/powershell_fileserver_powerbi/blob/main/imagens/Criar_chave_de_acesso.png?raw=true)</p>    

    + Escolha a opção Command line interface cli.
![image](https://github.com/j-a-vicente/powershell_fileserver_powerbi/blob/main/imagens/Command_line_interface_cli.png?raw=true)</p>    

    + Digite a justificativa para criação da credencial.
![image](https://github.com/j-a-vicente/powershell_fileserver_powerbi/blob/main/imagens/Criando_chave_de_acesso.png?raw=true)</p>        

    + Copie os valores "Chave de acesso" e "Chave de acesso secreta"
![image](https://github.com/j-a-vicente/powershell_fileserver_powerbi/blob/main/imagens/Chave.png?raw=true)</p>            

+ No container  digite o comando ```aws configure ```
    + AWS Access Key ID [None]: Digite a chave de acesso 
    + AWS Secret Access Key [None]: Digite a Chave de acesso secreta.
    + Default region name [None]: sa-east-1
    + Default output format [None]: json

Pronto o AWSClin está instalado e configurado com sucesso.

#### 08 - Baixando os logs para o container .

Para fazer o download dos logs é preciso está com o Autenticação multifator (MFA) configurado, o identificador do MFA. <b>arn:aws:iam::xxxxxxxxxxxxxxx:mfa/xxxxxxx</b>

+ Configurando o tocke temporário do MFA, o ```--token-code``` é número gerado pelo aplicativo de Autenticação.
```aws sts get-session-token --serial-number arn:aws:iam::xxxxxxxxxxxxxxx:mfa/xxxxxxx --token-code xxxxxx```
Observção: para criar um novo tocke é preciso fechar a conexão com o container  e abrir uma nova conexão.

O resultado será semelhante a este:
```
{
    "Credentials": {
        "AccessKeyId": "ASIAR3HUVZT6VPIOX6BQ",
        "SecretAccessKey": "K574I24G0+OrR2lQdVKKDf4mzBZ74u8XEuKhHJ3v",
        "SessionToken": "FwoGZXIvYXdzELj//////////wEaDEO4qPZtbdTrrgJNVSKGAczuFKafnv1deqrCMQnXxNXVnVB0G5HwvG6N1MVKf7hyfbt8lP9TQCT8ex75co56NUQuARz2wWjaIAtdvnFzB+4bBbreV1aUWdaNxxrgVjp51t2Z5SslShyua1UIHJT+lcm4G/Ckcx8+KvLoZ2OKf+yCqR1yuMX1zR5KkpXoJ8awy10T2J+rKJ+y86YGMig4VynYpoBQB44RWkX1gtKOxSrK7agKXFd9AvEzTCoendYyO2TMjirF",
        "Expiration": "2023-08-17T02:11:43Z"
    }
}
```
+ Exporte para o container  com os comandos.
```
export AWS_ACCESS_KEY_ID=ASIAR3HUVZT6VPIOX6BQ
export AWS_SECRET_ACCESS_KEY=K574I24G0+OrR2lQdVKKDf4mzBZ74u8XEuKhHJ3v
export AWS_SESSION_TOKEN=FwoGZXIvYXdzELj//////////wEaDEO4qPZtbdTrrgJNVSKGAczuFKafnv1deqrCMQnXxNXVnVB0G5HwvG6N1MVKf7hyfbt8lP9TQCT8ex75co56NUQuARz2wWjaIAtdvnFzB+4bBbreV1aUWdaNxxrgVjp51t2Z5SslShyua1UIHJT+lcm4G/Ckcx8+KvLoZ2OKf+yCqR1yuMX1zR5KkpXoJ8awy10T2J+rKJ+y86YGMig4VynYpoBQB44RWkX1gtKOxSrK7agKXFd9AvEzTCoendYyO2TMjirF
```

+ O comando abaixa é utilizado para baixar todos os logs gerado pelo RDS.
    + "--db-instance-identifier" é o nome da instância de banco.
    + "/pgbadger/producao" local que será salvo os logs.
    + Comando:
```
for filename in $( aws rds describe-db-log-files --db-instance-identifier db-xxx-xxxx-prd | jq -r '.DescribeDBLogFiles[] | .LogFileName' )
do
aws rds download-db-log-file-portion --db-instance-identifier db-xxx-xxxx-prd  --output text --no-paginate --log-file $filename  >> /pgbadger/producao/$filename
done
```
+ Processando os logs
    + ```cd /pgbadger/producao/error```
    + ls -l
    + pgbadger postgresql.log.* > index.html
+ No final do processo será gerado um arquivo ```index.html ```


#### 09 - Instalando e configurando o Apache.
```
apt-get update
apt-get install apache2 -y
apt-get install ufw -y
ufw app list
ufw allow 'Apache'
ufw status
systemctl status apache2
mkdir /var/www/pgbadger
chown -R $USER:$USER /var/www/pgbadger
chmod -R 755 /var/www/pgbadger

Copiar o arquivo out.html do pgbadger para o local /var/www/pgbadger/ renomear o arquivo para index.html
vim /etc/apache2/sites-available/pgbadger.conf
<VirtualHost *:80>
    ServerAdmin webmaster@localhost
    ServerName pgbadger
    ServerAlias www.pgbadger
    DocumentRoot /var/www/pgbadger
    ErrorLog ${APACHE_LOG_DIR}/error.log
    CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>

a2ensite pgbadger.conf
a2dissite 000-default.conf
apache2ctl configtest
systemctl restart apache2
```

#### 10 - Subindo pg_badger no Apache.

+ Copiar o relatório para Apache.
```cp /pgbadger/producao/error/index.html /var/www/pgbadger```

+Reiniciando o Apache
```
a2ensite pgbadger.conf
a2dissite 000-default.conf
apache2ctl configtest
systemctl restart apache2
```
#### 11 - Acessando pg_badger na estão de trabalho.
Na estação de trabalho entre no browser digite o endereço ```http://localhost:8585/index.html ```
