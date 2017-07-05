# Documento para instalar e integrar Katello e PuppetDB

- Seguido a instalação padrão do Katello, foi instalado 3.4, vide documentação de instalação abaixo.

  https://theforeman.org/plugins/katello/3.4/installation/index.html

- Neste procedimento foi utilizado o duas VM's;
  - CentOS 7 (Katello), katello.example.local;
  - CentOS 7 (PuppetDB) puppetdb.example.local;
- Realizar os procedimentos abaixo para instalar o PuppetDB com o Katello;

#### 1. Realizado o update do servidor e instalado o repositório de PC1 do puppet no servidor PuppetDB.
```shell
yum update -y
yum install https://yum.puppetlabs.com/el/7/PC1/x86_64/puppetlabs-release-pc1-1.1.0-5.el7.noarch.rpm -y
```

#### 2. Configurado o servidor de PuppetDB como agent do Katello.
```shell
yum install puppet -y 
puppet config set server katello.example.local
puppet agent -t 
```

#### 3. Precisamos assinar o certificado do puppetdb no katello.
```shell
puppet cert sign puppetdb.example.local
```

#### 4. Precisamos instalar o postgresql para cofigurar o PuppetDB
```shell
rpm -ivh --force https://download.postgresql.org/pub/repos/yum/9.5/redhat/rhel-7-x86_64/pgdg-redhat95-9.5-2.noarch.rpm
yum -y install postgresql95-server postgresql95-contrib
/usr/pgsql-9.5/bin/postgresql95-setup initdb
systemctl start postgresql-9.5
systemctl enable postgresql-9.5
```
#### 5. Precisamos ajustar a configuração do postgresql para aceitar as conexões do PuppetDB com o Banco de Dados, no arquivo  /var/lib/pgsql/9.5/data/pg_hba.conf precisamos adicionar a linha abaixo.
```ini
local     puppetdb       puppetdb                     trust
```
#### 6. Precisamos reiniciar o postgres para aplicar as novas configurações.
```shell
systemctl restart postgresql-9.5
```
#### 7. Após reiniciar o serviço do postgres, precisamos criar a database e permissionar.
```shell
su postgres
createdb puppetdb
createuser -a -d -E -P puppetdb
<SENHA>
<REPITIR SENHA>
exit
```
#### 8. Podemos testar a conexão com a database.
```shell
psql -U puppetdb -W
```
#### 9. Com sucesso na conexão com o banco, podemos instalar o PuppetDB.
```shell
puppet resource package puppetdb ensure=latest
```

#### 10. Após instalarmos o puppetdb, precisamos configurar a conexão com o banco de dados, configurado no passo 7, vamos configurar no arquivo: /etc/puppetlabs/puppetdb/conf.d/database.ini.
##### OBS: Caso esteja utilizando um banco externo os valores, subname deve apontar para o Bando de dados externo e sua respectiva database,
```ini
[database]
classname = org.postgresql.Driver
subprotocol = postgresql
 
# The database address, i.e. //HOST:PORT/DATABASE_NAME
subname = //localhost:5432/puppetdb
 
# Connect as a specific user
username = puppetdb
 
# Use a specific password
password = <SENHA DIGITADA NO PASSO 7>
```

#### 11. Também precisamos configurar o acesso ao dashboard do puppetdb, para realizar esse acesso precisamos configurar o bind em todos os ips, por padrão o jetty faz somente localhost. /etc/puppetlabs/puppetdb/conf.d/jetty.ini
```ini
[jetty]
# IP address or hostname to listen for clear-text HTTP. To avoid resolution
# issues, IP addresses are recommended over hostnames.
# Default is `localhost`.
host = 0.0.0.0
```

#### 12. Para a comunicação entre o Katello e o PuppetDB ocorrer, precisamos configurar o SSL para o puppetdb, para isso executamos o comando abaixo.
```shell
puppetdb ssl-setup
```

#### 13. Após configurar o banco, bind do jetty e configurar os certificados do puppetdb, precisamos reiniciar o serviço do puppetdb.
```shell
systemctl restart puppetdb
```

#### 14. No servidor katello, precisamos configurar o plugin do puppetdb ao foreman.
```shell
yum install tfm-rubygem-puppetdb_foreman -y
katello-service restart
```

#### 15. Após reiniciar o serviço teremos alguns campos para configurar pela interface.
Administer > Settings > PuppetDB.

Alteramos os campos puppetdb_address, puppetdb_dashboard_address, para o endereço do nosso puppetdb, neste caso puppetdb.example.local.

Alteramos para 'yes' o item puppetdb_enable.

#### 16. Após realizar a integração da interface, precisamos configurar a integração do puppet do katello com o puppetdb, para isso precisamos seguir os itens abaixo, no servidor de katello instalar o puppetdb-termini
```shell
puppet resource package puppetdb-termini ensure=latest
```

#### 17. Precisamos criar o arquivo /etc/puppetlabs/puppet/puppetdb.conf com o conteúdo abaixo, deve-se ajustar a url conforme o hostname da máquina que o puppetdb foi configurado, neste caso puppetdb.example.com
```ini
[main]
server_urls = https://puppetdb.example.com:8081
```

#### 18. No arquivo do puppet.conf do katello, precisamos adicionar as linhas abaixo.
```ini
reportstore = /var/log/puppetlabs/puppet
storeconfigs = true
storeconfigs_backend = puppetdb
```
E substituir a linha abaixo
```ini
reports = foreman
```
Por
```ini
reports = foreman, puppetdb
```

#### 19. Precisamos configurar o routes.yaml, o arquivo /etc/puppetlabs/puppet/routes.yaml com o conteúdo abaixo.
```ini
---
master:
  facts:
    terminus: puppetdb
    cache: yaml
```

#### 20. Para termos certeza que as permissões estejam corretas, podemos rodar o comando abaixo
```shell
chown -R puppet:puppet `puppet config print confdir
```

#### 21. Reinicializamos os serviços para termos certeza que as configurações realizadas serão aplicadas.
No Katello.
```shell
katello-service restart
```
No PuppetBD
```shell
systemctl restart puppetdb
```

#### 22. Após os serviços ficarem disponíveis, podemos realizar os testes com o puppetdb e o katello, podemos utilizar um exported resources de file, conforme o exemplo abaixo que temos uma classe para exportar o resouce file.
Classe de exportação.
```puppet
class files::export_file {
 
  @@file {"/root/${::hostname}":
    ensure => file,
    content => "My hostname is ${::hostname}"
  }
 
}
```
Classe para importação.
```puppet
class files::import_file {
 
  File <<| |>>
 
}
```

#### 23. Após criação das classes e importação no katello, podemos aplicar pela GUI a classe e provar o funcionamento do exported resources com o Katello.

#### Documentações Utilizadas.
http://blog.aeciopires.com/instalando-configurando-puppetdb-puppetexplorer-puppetboard/

https://github.com/theforeman/puppetdb_foreman

https://theforeman.org/plugins/katello/3.4/index.html

https://docs.puppet.com/puppet/4.10/config_file_routes.html

https://docs.puppet.com/puppet/4.10/lang_exported.html
