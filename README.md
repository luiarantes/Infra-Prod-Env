# Ambiente de Produção AWS Elastic Beanstalk para Aplicação PHP e MySql 

**Descrição:** O Objetivo é ter ao final um ambiente totalmente operacional utilizando Elastic Beanstalk e Banco de Dados MySql no RDS.

**Requisitos:**

Sistema Operacional: **Ubuntu Server 20.04 LTS**

**Pacotes:**

* vim
* git
* unzip
* mysql-client-core-8.0
* aws cli
* eb cli

### Preparando o ambiente

##### Atualizando sistema operacional, instalando requisitos e reinicializando

```
$ sudo apt update
$ sudo apt upgrade -y
$ sudo apt install vim git unzip mysql-client-core-8.0 -y

$ sudo shutdown -r now
```

##### Instalando AWS CLI

```
$ mkdir ~/Install ; cd ~/Install
$ curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64-2.0.30.zip" -o "awscliv2.zip"
$ unzip awscliv2.zip
$ sudo ./aws/install
```

Referência:
[https://docs.aws.amazon.com/pt_br/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-install](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/install-cliv2-linux.html#cliv2-linux-install)

##### Configurando AWS CLI

Para ceder acesso do AWS CLI ao ambiente será preciso criar o ID da chave de acesso e a chave de acesso secreta no console da AWS seguindo os passos do link abaixo:

###### Obs: O usuário do IAM para qual será gerada a chave precisa ter permissão administrativa na console AWS

[https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-creds)

```
$ aws configure

AWS Access Key ID [None]: AccessKeyID
AWS Secret Access Key [None]: SecretAccessKey
Default region name [None]: us-east-1
Default output format [None]: json

Para testar vamos listar as zonas de disponibilidade com o seguinte comando: 

$ aws ec2 describe-availability-zones | grep ZoneName
```

Referência
[https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config](https://docs.aws.amazon.com/pt_br/cli/latest/userguide/cli-configure-quickstart.html#cli-configure-quickstart-config)

##### Instalando AWS EB CLI

```
$ curl -O https://bootstrap.pypa.io/get-pip.py
$ python3 get-pip.py --user
$ echo "export PATH=~/.local/bin:\$PATH" >> ~/.bashrc
$ source ~/.bashrc
$ pip install awsebcli --upgrade --user
```

### Criando Infraestrutura de Produção na AWS

#### Banco de Dados MySqs no RDS

```
$ aws rds create-db-instance \
--db-instance-identifier mysql-inst-01 \
--allocated-storage 30 \
--db-instance-class db.t2.micro \
--engine mysql \
--master-username root \
--master-user-password mysql.rootp4ss \
--auto-minor-version-upgrade \
--db-name mysqldb \
--no-multi-az \
--preferred-maintenance-window sun:01:00-sun:03:00 \
--publicly-accessible \
--backup-retention-period 3
```
###### Obs: Em um ambiente real é sempre importante proteger a instância com o parâmetro --deletion-protection, não iremos utilizar para facilitar a remoção do banco ao final.

Referência:
[https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html](https://docs.aws.amazon.com/cli/latest/reference/rds/create-db-instance.html)

##### Identificando caminho para a instância de banco de dados no RDS

```
$ aws rds describe-db-instances --db-instance-identifier mysql-inst-01 | grep Address
```

##### Conectando ao banco de dados, criando tabela e inserindo registros
```
$ mysql -u root -h [host identificado com o comando aws rds describe-db-instances] -p
Enter password: mysql.rootp4ss

mysql> USE mysqldb;

mysql> CREATE TABLE tabela (id int(11) NOT NULL AUTO_INCREMENT, coluna varchar(255) NOT NULL, PRIMARY KEY (id));

mysql> INSERT INTO tabela (coluna) VALUES ('Hello World');

mysql> exit
```

#### Elastic Beanstalk

##### Criando role para permitir acesso do Elastic Beanstalk ao EC2

1. Acessar o AWS Console
1. Entrar no IAM / Role no link abaixo

[https://console.aws.amazon.com/iam/home?region=us-east-1#/roles](https://console.aws.amazon.com/iam/home?region=us-east-1#/roles)

1. Botão azul "Create Role"
1. Selecionar serviço e caso de uso EC2
1. Pesquisar: "ElasticBeanstalkFullAccess" + Selecionar checkbox
1. Next: Tags + Next: Review
1. Role name "aws-elasticbeanstalk-ec2-role"
1. Create Role

##### Criando aplicação

```
$ aws elasticbeanstalk create-application --application-name HelloWorld --description "HelloWorld"
```

Referência: [https://docs.aws.amazon.com/cli/latest/reference/elasticbeanstalk/create-application.html](https://docs.aws.amazon.com/cli/latest/reference/elasticbeanstalk/create-application.html)

##### Verificando disponibilizade de domínio

```
$ aws elasticbeanstalk check-dns-availability --cname-prefix hello-world-8373647565
```

# Criando ambiente

```
- Criando diretório para arquivos de configuração 

$ mkdir -p ~/eb ; touch ~/eb/options.txt ; cd ~/eb/

```

- Editar arquivo options.txt adicionando a seguinte configuração para o autoscaling com instâncias tipo t3.nano

```
$ vim ~/eb/options.txt

[
    {
        "Namespace": "aws:autoscaling:launchconfiguration",
        "OptionName": "IamInstanceProfile",
        "Value": "aws-elasticbeanstalk-ec2-role"
    },
    {
        "Namespace": "aws:autoscaling:launchconfiguration",
        "OptionName": "EC2KeyName",
        "Value": "web-key-pair"
    },
    {
        "Namespace": "aws:autoscaling:launchconfiguration",
        "OptionName": "InstanceType",
        "Value": "t3.nano"
    },
    {
        "Namespace": "aws:autoscaling:launchconfiguration",
        "OptionName": "SecurityGroups",
        "Value": "default"
    }
]
```

##### Executando a criação do ambiente

```
$ aws elasticbeanstalk create-environment \
--cname-prefix hello-world-8373647565 \
--application-name HelloWorld \
--environment-name HelloWorld-Prod \
--solution-stack-name "64bit Amazon Linux 2 v3.1.2 running PHP 7.4" \
--option-settings file://options.txt
```

Referência: [https://docs.aws.amazon.com/pt_br/elasticbeanstalk/latest/dg/environments-create-awscli.html](https://docs.aws.amazon.com/pt_br/elasticbeanstalk/latest/dg/environments-create-awscli.html)

### Deploy de aplicação

```
$ cd ~
$ git clone https://github.com/luiarantes/PHP-MySql-HelloWorld.git
$ cd ~/PHP-MySql-HelloWorld
$ eb init

  1) us-east-1 : US East (N. Virginia)

  1) HelloWorld

Do you wish to continue with CodeCommit? (Y/n): n
```
Comandos para verificar o status e saúde do ambiente

```
$ eb status
$ eb health
```

Resultado esperado antes de realizar o deploy

* Status: Ready
* Health: Green

##### Realizando o deploy

```
$ eb deploy
```
O deploy foi efetuado porém não existe conexão com o banco de dados de produção, pois o arquivo com as credenciais não pode ser armazenado no repositório remoto, vamos criar o arquivo agora e depois realizar um novo deploy

### Alterar caminho para conexão com banco de dados

```
$ cd ~/PHP-MySql-HelloWorld
$ _DB=$(aws rds describe-db-instances --db-instance-identifier mysql-inst-01 | grep Address | awk '{ print $2 }' | sed 's/,//' | sed 's/\"//' | sed 's/"//')
$ echo "<?php \$HOST=\"$_DB\"; \$USER=\"root\"; \$PASS=\"mysql.rootp4ss\"; \$DB=\"mysqldb\"; ?>" > ~/PHP-MySql-HelloWorld/db-vars.php
```

##### Definir nome de usuário e endereço de e-mail

```
$ git config --global user.name "Name"
$ git config --global user.email "email@example.com"
$ echo ".gitignore" >> .gitignore
```

##### Realizando deploy de atualização

```
$ git add db-vars.php
$ git commit -m "Add Host MySql"
$ eb deploy
```

##### Acessar ambiente

[http://hello-world-8373647565.us-east-1.elasticbeanstalk.com](http://hello-world-8373647565.us-east-1.elasticbeanstalk.com)

##### Para remover todo o ambiente utilize os comandos abaixo:

```
$ aws rds delete-db-instance \
--db-instance-identifier mysql-inst-01 \
--skip-final-snapshot \
--delete-automated-backups

$ cd ~/PHP-MySql-HelloWorld
$ eb terminate

To confirm, type the environment name: HelloWorld-Prod

$ aws elasticbeanstalk delete-application --application-name HelloWorld
```
