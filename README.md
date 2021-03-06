# Ansible Role: SAP ASE

Esta *role* encapsula parte da toda lógica da criação de um servidor
*SAP Sybase ASE Express Edition for Linux* e foi baseada em [ansible-sap-iq](https://github.com/andrewrothstein/ansible-sap-iq).

Foi testado em:

 - Debian 9
 - Debian 10
 
Vídeo: https://youtu.be/y0dG2jR_NJQ

## Recursos
 
 - Seleciona a versão express
 - Cria um usuário e grupo sybase, deixando como sudo
 - Configura ulimit para unlimited
 - Instala dependências necessárias, como libaio1, unzip
 - Parte do pressuposto que você baixou o .tgz no servidor
 - Provê um arquivo *response* com uma parametrização básica para instalação do SAP Sybase ASE
 - Permite especificar o caminho do log de serviço do servidor
 - Configura o service *sybase* com start e stop
 - Service *sybase* inicializa automaticamente no boot
 - SAP ase server sobre na porta 5000 e backup server na porta 5001
 - As variáveis de ambiente para usar o isql estão globais para todos usuários do sistema
 - Script de backup para um banco de dados

# Instalação

O *response file* usado neste role está disponível em *templates/response.j2*
e por padrão habilitar o servidor SAP ASE e o servidor de Backup. Para
habilitar outros serviços, esse arquivo pode ser alterado.

Nesta instalação, vamos isolar a partição do sap ase
deixando um diretório para os dados e outra para log:
 */replicado* e */replicado/log*, assim, especifique as
variáveis do ansible correspondente:

    sap_ase_home: '/replicado'
    sap_ase_log: '/replicado/log'

Baixar o arquivo *.tgz* oficial do sybase e colocá-lo no seu servidor.
Especifique a variável do ansible correspondente:

    sap_ase_tar: '/root/ASE_Suite.linuxamd64.tgz'

Segue o playbook completo com as demais variáveis que devem ser configuradas:

    - name: deploy
      become: yes
      hosts: sybase

      tasks:

        - name: fflch.sap-ase
          include_role:
            name: fflch.sap-ase
          vars:
            sap_ase_home: '/replicado'
            sap_ase_log: '/replicado/log'
            sap_ase_tar: '/root/ASE_Suite.linuxamd64.tgz'
     
            sap_ase_host: 'replicado.fflch.usp.br'
            sap_ase_password: "SenhaDoServidor123"

            sap_ase_backup_enable: True
            sap_ase_backup_database: 'fflch'

Seu servidor foi instalado em /replicado/sap.

# Algumas configurações pós-instalação

Acessar servidor sybase estando conectado via ssh,
com o usuário *sa*.
O *-w1000* torna os outputs mas *bonitos*:

    isql -Usa -PSUA_SENHA -SSYBASE -w1000

Acessar servidor sybase remotamente, com usuário *sa*, 
dado que você tem o isql localmente:

    isql -Usa -PSUA_SENHA -S192.168.8.56 -w1000

(recomendação replicado) Ver e alterar o número de locks:

    1> sp_configure 'number of locks'
    2> go
    1> sp_configure 'number of locks', 100000 
    2> go
    
(recomendação replicado) Ver e alterar o número de conexões:

    1> sp_configure 'number of user connections'
    2> go
    1> sp_configure 'number of user connections', 150
    2> go

(recomendação replicado) Criar usuário dbmaint.

    1> use master
    2> go
    1> sp_addlogin "dbmaint", "SuaSuperSenha123"
    2> go

Lista todos usuários:

    1> use master
    2> go
    1> select name from syslogins
    2> go

A versão express nos permite usar apenas 50G 
(Exceed the limit of 51200 MB for ASE Express Edition)
Se você usou o response file desta *role*, foi configurado 
os seguintes discos/devices, entre outros:

    master: 200 MB
    tempdbdev: 2GB
    sysprocsdev: 500MB
    systemdbdev: 500MB

Reservando então 5G para os devices do próprio sistema, podemos
usar os 45GB restantes. No nosso caso, vamos criar um servidor
alvo para replicação e vamos criar um device para *data* e 
outro para *log*.

(recomendação replicado) Criando disco de 20G para dados, que chamaremos de *fflch_data_01*:

    1> use master
    2> go

    1> disk init 
    name = "fflch_data_01", 
    physname = "/replicado/sap/data/fflch_data_01.dat", 
    size = "20G"
    5> go

Caso precise deletar o disk/device:

    1> sp_dropdevice fflch_data_01
    2> go
    rm /replicado/sap/data/fflch_data_01.dat

Para log vamos criar um device de 3GB que chamaremos de *fflch_log_01*:

    1> use master
    2> go
    
    1> disk init
    name = "fflch_log_01", 
    physname = "/replicado/log/fflch_log_01.dat", 
    size = "3G"
    5> go 
    
Criando banco de dados nos discos criados e alocando todo espaço disponível:

    1> CREATE DATABASE fflch ON fflch_data_01='20G' LOG ON fflch_log_01='3G'
    2> go

Verificar os *devices* criados:

    1> sp_helpdevice
    2> go

Listar bancos de dados:

    1> sp_helpdb	
    2> go
    1> sp_helpdb fflch	
    2> go

Caso necessário, pode-se criar novos devices no futuro,
como *fflch_data_02* e *fflch_log_02* e adicioná-los no database fflch
assim:

    ALTER DATABASE fflch ON fflch_data_02='20G'
    ALTER DATABASE fflch LOG ON fflch_log_02='3G'

(recomendação replicado) Ativar a opção de truncate log on 
checkpoint para o banco de dados fflch:

    1> use master
    2> go
    1> sp_dboption fflch, "trunc log on chkpt", true
    2> go

(recomendação replicado) Ativar a opção de *select into* para o banco de dados fflch:

    1> use master
    2> go
    1> sp_dboption fflch, "select into", true
    2> go

(recomendação replicado) Deixar usuário dbmaint com privilégio de 
criar tabelas no banco de dados *fflch* (alias de dbo):

    1> use fflch
    2> go
    1> sp_addalias dbmaint,dbo
    2> go
    1> sp_helpuser
    2> go

## Dicas de uso geral

Mostrar tabelas do banco de dados fflch:

    1> use fflch
    2> go
    1> exec sp_tables '%', '%', 'fflch',"'TABLE'"
    2> go

Outra forma de mostrar tabelas do banco de dados fflch:

    1> use fflch
    2> go 
    1> select name from sysobjects where type = 'U' or type = 'P'
    2> go

Mostra em qual banco de dados que estou connectado no momento:

    1> select db_name()
    2> go

Carregar um arquivo sql:

    isql -Usa -PSuaSenha -SYBASE -iSeuArquivoComSql.sql
       
Dicas de IDE para fazer queries graficamente no banco: 

 - https://dbeaver.io

Criando um usuário *fulano* com senha:

    1> use master
    2> go
    1> sp_addlogin "fulano", "SUA_SENHA"
    2> go

Adicionar usuário acima ao banco fflch:

    1> use fflch
    2> go
    1> sp_adduser 'fulano', 'fulano'
    2> go

Testar conexão do novo usuário:

    isql -Ufulano -PSUA_SENHA -SSYBASE -w1000

Gerar SQLs, que podem ser copiadas e aplicadas, que darão 
permissão de SELECT em todas tabelas do banco fflch:

    1> use fflch
    2> go
    1> select 'grant select on ' + name + ' to fulano' from sysobjects where type = 'U' or type = 'P'
    2> go

Gerar um dump do banco de dados:

    isql -Usa -PSuaSenha -SYBASE -w1000
    1> use master
    2> go
    1> dump database fflch to "/replicado/fflch.dump"
    2> go

Carregar banco de dados a partir de um dump:

    isql -Usa -PSuaSenha -SYBASE -w1000
    1> use master
    2> go
    1> load database fflch from "/replicado/fflch.dump"
    2> go
    1> online database fflch
    2> go
