# Installer SonarQube sur une EC2 AWS sous Ubuntu 18.04 LTS

/ ! \\ Ne fonctionne pas avec AWS t2.micro qui ne sont pas assez puissances

# SSL

Pour le SSL, une fois toute la configuration effectuée, voir le lien suivant: 

## Prerequisites & Java installation - 11 is needed for the last SonarQube version

```
sudo apt update
sudo apt upgrade -y
sudo apt-get install -y software-properties-common
sudo apt install openjdk-11-jdk -y
java -version
sudo apt-get install unzip
```

## PostgreSql Installation

```
sudo sh -c 'echo "deb http://apt.postgresql.org/pub/repos/apt/ `lsb_release -cs`-pgdg main" >> /etc/apt/sources.list.d/pgdg.list'
wget -q https://www.postgresql.org/media/keys/ACCC4CF8.asc -O - | sudo apt-key add -
sudo apt-get -y install postgresql postgresql-contrib

# Add secured User to the OS
sudo adduser sonar 

# PostgreSql Configuration for SonarQube
sudo passwd postgres
su - postgres
createuser sqube
psql
ALTER USER sqube WITH ENCRYPTED password 'myMdp123!';
CREATE DATABASE sqube OWNER sqube;
\q
exit
```

## SonarQube Installation

```
sudo wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-8.2.0.32929.zip
sudo unzip sonarqube-8.2.0.32929.zip

# SonarQube Server Configuration to use PostgreSql Database
sudo nano sonarqube-8.2.0.32929/conf/sonar.properties
	sonar.jdbc.username=sqube
	sonar.jdbc.password=myMdp123!
	sonar.jdbc.url=jdbc:postgresql://localhost/sqube
	sonar.web.host=0.0.0.0
	sonar.ce.javaAdditionalOpts=-server
	
# SonarQube Security, Run as sonar user with limited rights
sudo nano sonarqube-8.2.0.32929/bin/linux-x86-64/sonar.sh
	RUN_AS_USER=sonar
sudo nano sonarqube-8.2.0.32929/elasticsearch/config/elasticsearch.yml
	node.name:${hostname}
	network.host: 0.0.0.0
sudo visudo
	root ALL=(ALL:ALL) ALL
	sonar ALL=(ALL) NOPASSWD: ALL

# SonarQube files, move to Optional folder	
sudo mkdir /opt/sonar/
sudo chown -R sonar:sonar /opt/sonar
sudo mv sonarqube-8.2.0.32929 /opt/sonar/	
cd /opt/sonar/
sudo chown -R sonar:sonar /opt/sonar

# Configure SonarQube - extend allowed virtual memory
su - sonar
sudo sysctl -w vm.max_map_count=262144

# Start SonarQube and check if it is running as expected using ports 9000 and 9001.
/opt/sonar/sonarqube-8.2.0.32929/bin/linux-x86-64/sonar.sh start
sudo netstat -plnt
# If any error, Check the logs
# cat /opt/sonar/sonarqube-8.2.0.32929/logs/sonar.log
# cat /opt/sonar/sonarqube-8.2.0.32929/logs/es.log
# cat /opt/sonar/sonarqube-8.2.0.32929/logs/web.log
# cat /opt/sonar/sonarqube-8.2.0.32929/logs/access.log
```

## Apache minimal Installation and Configuration

```
# Apache installation and configuration
sudo apt-get install apache2 -y
sudo a2enmod proxy
sudo a2enmod proxy_http

# Apache creation of a website for SonarQube
sudo nano /etc/apache2/sites-available/sonar.conf
<VirtualHost *:80>
	ServerName sub.domain.com
	ServerAdmin admin@example.com
	ProxyPreserveHost On
	ProxyPass / http://127.0.0.1:9000/
	ProxyPassReverse / http://127.0.0.1:9000/
	TransferLog /var/log/apache2/sonarm_access.log
	ErrorLog /var/log/apache2/sonar_error.log
</VirtualHost>

# apache and the newly created site
sudo a2ensite sonar
sudo systemctl restart apache2
```

### Redémarrer Apache et le serveur sonarqube

```
#Restart servers
sudo service apache2 restart
/opt/sonar/sonarqube-8.2.0.32929/bin/linux-x86-64/sonar.sh restart
```

### Logs Utiles

```
#Usefull Logs
journalctl | tail
systemctl status apache2.service
journalctl -xe
tail -f /var/log/apache2/sonarm_access.log
#lets encrypt logs
/var/log/letsencrypt
```

# Configuration AWS de la EC2

## Inbound Rules

SSH: 22

HTTP: 80

HTTPS: 443

Accès Sonar-Scanner: 9000

## Rôles

Ajouter / modifier un rôle pour autoriser Code Build à lire les valeurs de Secrets Manager sur les bonnes resources:

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetResourcePolicy",
                "secretsmanager:GetSecretValue",
                "secretsmanager:DescribeSecret",
                "secretsmanager:ListSecretVersionIds"
            ],
            "Resource": [
                "arn:aws:secretsmanager:ca-central-1:545456465465:secret:stage/sonar-IfKMVF",
                "arn:aws:secretsmanager:ca-central-1:545456465465:secret:dev/sonar-bAEKiI",
                "arn:aws:secretsmanager:ca-central-1:545456465465:secret:prod/sonar-cIyZAC"
            ]
        },
        ........
```

# Optionnel - Installer Sonar Scanner sur les EC2 Linux

Si l’on veut pouvoir lancer des exécutions de Sonar-Scanner manuellement depuis les l’environnement ElasticBeankStalk, c’est possible

```
wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.2.0.1873-linux.zip
unzip sonar-scanner-cli-4.2.0.1873-linux.zip
sudo mv sonar-scanner-4.2.0.1873-linux /opt/sonar-scanner-4.2.0.1873-linux
rm sonar-scanner-cli-4.2.0.1873-linux.zip
export PATH="$PATH:/opt/sonar-scanner-4.2.0.1873-linux/bin"
sudo nano .bashrc
	export PATH="$PATH:/opt/sonar-scanner-4.2.0.1873-linux/bin"
sonar-scanner -h
# pour info: sudo nano /opt/sonar-scanner-4.2.0.1873-linux/conf/sonar-scanner.properties
```

# Optionnel - Installer Sonar-Scanner pour Windows

Télecharger le zip [https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/)

Extraire dans Program Files

Ajouter le chemin vers Bin dans les variables d'environnement C:\\Program Files\\Sonar\\sonar-scanner-4.2.0.1873-windows\\bin

Ouvrir une nouvelle fenêtre de commandes:

```
sonar-scanner -h
```

# Configurer un Project Java pour intégrer l’exécution SonarQube via le pipeline AWS

## Fichier buildspec.yml

Editer le fichier **buildspec.yml** et ajouter les lignes suivantes:

Les variables à personnaliser sont

Avant de publier, valider le fichier YML ici: [http://www.yamllint.com/](http://www.yamllint.com/)

```
env:
  secrets-manager:
    SonarLogin: {{secretName}}:{{secretKey}}
    SonarHostUrl: {{secretName}}:{{secretKey}}
    SonarProjectKey: {{secretName}}:{{secretKey}}
pre_build:
  commands:	  
    - wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.2.0.1873-linux.zip
    - unzip ./sonar-scanner-cli-4.2.0.1873-linux.zip
    - export PATH=$PATH:/sonar-scanner-cli-4.2.0.1873-linux/bin/
build:
  commands:
    - mvn sonar:sonar -Dsonar.login=$SonarLogin -Dsonar.host.url=$SonarHostUrl -Dsonar.projectKey=$SonarProjectKey
    - sleep 5
    - curl http://{{ec2}}.ca-central-1.compute.amazonaws.com/api/qualitygates/project_status?projectKey=$SonarProjectKey >result.json
    - cat result.json
    - if [ $(jq -r '.projectStatus.status' result.json) = ERROR ] ; then $CODEBUILD_BUILD_SUCCEEDING -eq 0 ;fi
```

## Git

Ajouter dans le gitignore

```
.scannerwork/**
```

## sonar-project.properties

Pour plus de configuration, créer à la racine du projet un fichier: sonar-project.properties. Voici une configuration par exemple

```
# SOURCES
sonar.java.source=8
sonar.sources=src/main/java
sonar.java.binaries=target/classes
sonar.sourceEncoding=UTF-8
# EXCLUSIONS
# (exclusion of Lombok-generated stuff comes from the `lombok.config` file)
sonar.coverage.exclusions=**/*Exception.java , **/MyJavaApplication.java
# TESTS
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
sonar.junit.reportsPath=target/surefire-reports/TEST-*.xml
sonar.tests=src/test/java
```

# Sonar-Scanner

Différentes façon d’utiliser Sonar-Scanner manuellement:

Maven: mvn sonar:sonar

Windows/linux: sonar-scanner

Gradle

…

Plus d’informations ici: [https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/](https://docs.sonarqube.org/latest/analysis/scan/sonarscanner/)

# Configurer Jacoco pour le “Code Coverage” sur SonarQube

## Plugin Maven

Ajouter le plugin à Maven, dans le **pom.xml**

```
<plugin>
	<groupId>org.jacoco</groupId>
	<artifactId>jacoco-maven-plugin</artifactId>
	<version>${jacoco-maven-plugin.version}</version>
	<executions>
		<execution>
			<id>default-prepare-agent</id>
			<goals>
				<goal>prepare-agent</goal>
			</goals>
		</execution>
		<!-- Trying to get the report to be created on simple unit tests too. -->
		<execution>
			<id>default-unit-test-report</id>
			<phase>prepare-package</phase>
			<goals>
				<goal>report</goal>
			</goals>
		</execution>
		<!-- Overwrite unit test results once the integration tests are done. -->
		<execution>
			<id>default-integration-test-report</id>
			<phase>post-integration-test</phase>
			<goals>
				<goal>report</goal>
			</goals>
		</execution>
	</executions>
</plugin>
```

## Les exclusions:

Toujours dans le **pom.xml**, ajouter la propriété: sonar.coverage.exclusions

```
<project>....
  <properties>....
		<sonar.coverage.exclusions>
			**/*Exception.java,
			**/*Dto.java,
			**/MyApplication.java,
		</sonar.coverage.exclusions>
	</properties>
	......
</project>
```

## Générer le rapport Jacoco Manuellement.

Le résultat se trouvera dans **target/site/jacoco/\***

```
mvn jacoco:report
```

## Configurer SonarQube pour trouver le rapport Jacoco

```
# TESTS
sonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
sonar.junit.reportsPath=target/surefire-reports/TEST-*.xml
sonar.tests=src/test/java
```

## AWS, Pipelines: générer le rapport Jacoco automatiquement

Dans le fichier **Buildspec.yml** à la racine du projet, s’assurer de la présence des lignes:

```
version: 0.2
env:
  secrets-manager:
    SonarLogin: prod/sonar:SonarLogin
    SonarHostUrl: prod/sonar:SonarHostUrl
    SonarProjectKey: prod/sonar:SonarProjectKey
phases:
  install:
  pre_build:
    commands: 
      # Downloading SonarScanner
      - wget https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.2.0.1873-linux.zip
      - unzip ./sonar-scanner-cli-4.2.0.1873-linux.zip
      - export PATH=$PATH:/sonar-scanner-cli-4.2.0.1873-linux/bin/
  build:
    commands:
      # Executing Jacoco Analysis for Code Coverage
      - mvn jacoco:report
      # Executing SonarScanner
      - mvn sonar:sonar -Dsonar.login=$SonarLogin -Dsonar.host.url=$SonarHostUrl -Dsonar.projectKey=$SonarProjectKey
```
