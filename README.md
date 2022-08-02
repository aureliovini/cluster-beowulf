# Montagem e configuração de um cluster Beowulf

## Introdução
### O que é um cluster?
Um cluster é um grupo de dois, ou mais, computadores executados em paralelo. Esta característica
permite que tarefas com alta demanda computacional e que possam ser paralelizável
sejam distribuídas entre os nós deste cluster, podendo aproveitar o poder de processamento
combinado de cada nó.

### O computador
Neste projeto, nosso cluster será instalado em quatro Raspberry PI 3, onde inicialmente
detectou-se que dois deles não estavam ligando por algum problema em seus cartões micro SD
podendo ser na imagem (ISO) do sistema operacional (SO) gravada ou algum defeito no cartão.

## Equipamentos
- 4x Raspberry PI 3
- 1x Roteador TL-WR740N(BR) v4.20
- 8x Cabos Ethernet
- 16 GB SD card
- Case para cluster

## Instalando o Sistema Operacional
Os Raspberry PI que iremos utilizar possuíam inicialmente seu próprio sistema operacional
(SO), sendo assim baixamos no site do Ubuntu a versão server apropriada para o Raspberry PI
(Ubuntu Server 22.04 LTS) neste [link](https://ubuntu.com/download/raspberry-pi).

A maneira mais prática para formatação destes cartões foi inserindo-os em um pendrive
apropiado, capaz de acessar os dados de cada cartão, e com a utilização de um programa de
gerenciamento de disco padrão de um computador pessoal acessamos os dados de cada cartão
e o formatamos. Com este método, foi possível perceber que os dois cartões com problemas
possuíam a imagem corrompida do SO, uma vez que conseguimos acessar os seus dados e
formatá-los sem problema algum

Após todos os quatro cartões micro SD serem formatados e instalados com o Ubuntu Server
22.04 LTS, nosso próximo passo será configurar cada Raspberry PI. Para a criação de um cluster
Beowulf, iremos configurar um Raspberry PI como **Server** e os três restantes como **Nós**.

## Configurando o roteador
O roteador a ser utilizado terá foco exclusivo para a conexão da rede local (LAN), sendo
assim iremos desabilitar as opções de rede wireless e dhcp.

Para desabilitar as redes escolhidas, o roteador foi conectado à um computador auxiliar e
através de um navegador acessamos o ip **192.168.1.1** e desmarcando a checkbox **Habilitado**
nos menus **Wireless** e **DHCP**. Após desmarcar cada checkbox foi salva a configuração e
reiniciado o roteador.

Feito estes passos, obtemos agora um roteador dedicado à rede local.

## Configurando o servidor
Após a instalação do SO, efetuaremos os passos logado como **super usuário (su)**. Para
efetuarmos o login utilizaremos o comando a seguir
```
>> sudo su
```
Para garantir o funcionamento do cluster é necessário que a data e hora estejam equivalentes
entre todos os nós, tanto no master quanto nos slaves. Por isso, ao ligar o equipamento, deverá
ser digitado no terminal
```
>> date –set "DIA MES ANO HORA:MINUTO:SEGUNDO"
```
onde **MES** é a abreviado com três letras (e.g., JAN, FEV, MAR). Para conferir se as datas estão
equivalentes basta executar o comando
```
>> date
```

### Atualizando o SO
Antes de começar a configuração de nosso servidor, iremos atualizar o sistema operacional
executando os comandos a seguir
```
>> apt update
>> apt upgrade -y
```

### Configurando a placa de rede
Para a configuração da placa de rede, a fim de se obter um IP fixo para comunicação
LAN, é necessário a escolha do IP fixo desejado para cada máquina. Deverá editar o arquivo
**/etc/netplan/50-cloud-init.yaml** para que fique com as opções abaixo
```
network:
	version:2
	renderer:networkd
	ethernets:
		eth0:
			dhcp4: true
			addresses: [10.0.1.X/24]
			routes:
				- to: default
				  via: 10.0.1.1
			nameservers:
				addresses: [8.8.8.8, 1.1.1.1]
```
onde **X** será o IP do servidor.

Em seguida, no arquivo **/etc/hosts** iremos adicionar as linhas
```
127.0.0.1	localhost
10.0.1.5	servidor.cluster	servidor
10.0.1.6	no01.cluster		no01
10.0.1.7	no02.cluster		no02
10.0.1.8	no03.cluster		no03
```
Após editar os arquivos de configuração de rede, é necessário aplicar a alteração, para isto
basta dar o seguinte comando
```
>> netplan apply
```
A placa de rede está configurada para a conexão LAN a partir de agora.

### Instalando e configurando o protocolo NFS
Com o sistema operacional atualizado, iremos instalar e configurar o **protocolo NFS**, que
permitirá que um usuário em um computador cliente acesse arquivos em uma rede da mesma
forma que acessaria um arquivo de armazenamento local.

Para instalar o pacote responsável ao protocolo NFS para o servidor basta executar o seguinte
comando
```
>> apt install nfs-kernel-server -y
```
Iremos criar o diretório **/mnt/nfsshare** para ser compartilhado, para isto iremos utilizar o
seguinte comando
```
>> mkdir /mnt/nfsshare
```
Agora, atribuiremos os direitos corretos à pasta que queremos compartilhar na rede. Basta
usar os três comandos a seguir
```
>> chown -R usuario:usuario /mnt/nfsshare
>> find /mnt/nfsshare/ -type d -exec chmod 755
>> find /mnt/nfsshare/ -type f -exec chmod 644
```
O primeiro comando irá fornecer o direito à dono de todos os arquivos e pastas contidos no
diretório ao usuário e ao grupo “usuario” selecionado.

O segundo comando irá procurar todos os diretórios dentro da pasta “/mnt/nfsshare” e
executa o comando chmod para dar a cada diretório a permissão **755**.

O terceiro comando procura todos os arquivos dentro do diretório e aplica as permissões
**644** ao arquivo.

O número de três dígitos referente às permissões podem ser analisadas como uma combinação
de três números onde o numeral da centena é a permissão referente ao dono, o numeral da
dezena é a permissão referente ao grupo e o numeral das unidades é a permissão referente
a outros grupos. Dúvidas sobre o significado de cada número e suas respectivas permissões
podem ser observadas na tabela a seguir.

| Índice   |Permissão      |
|----------|:-------------:|
|0         |-rwx 			|
|1         |+x				|
|2         |+w				|
|3         |+wx				|
|4         |+r				|
|5         |+rx				|
|6         |+rw				|
|7         |+rwx			|

Onde **r** é permissão para a leitura de um arquivo, **w** é a permissão para edição de um arquivo
e **x** a permissão de execução de um arquivo (ou acesso através do comando **cd** caso seja um
diretório). E os sinais **+** ou **-** indicam o incremento ou decremento da permissão informada.

O próximo passo será modificar o arquivo exports, para isto iremos executar o seguinte
comando
```
>> vi /etc/exports
```
Neste ponto, o arquivo selecionado irá abrir no terminal e iremos inserir o texto abaixo na
última linha e logo após basta salvar as alterações.
```
/mnt/nfsshare *(rw)
/home *(rw)
```
Esta linha informa que desejamos compartilhar os diretórios **/mnt/nfsshare** e **/home**, o
asterisco * define que todos os endereços IP’s da conexão tem permissão para acessar este
compartilhamento e o **rw** permite solicitações de leitura e gravação no volume NFS.

Para atualizar a tabela atual de exportações disponível para o servidor NFS basta executar o
comando
```
>> exportfs -ra
```
Feito estes passos, temos agora o protocolo NFS instalado e configurado corretamente para
o Raspberry Pi Server.

### Instalando e configurando o protocolo NIS
Para instalar o NIS server, o comando será
```
>> apt install nis -y
```
Após a instalação do NIS, é necessário definir o **domainname**

Para o protocolo NIS, o **ypserv** aguarda pedidos de consulta para dar aos NIS clientes
respostas. Como este protocolo é um serviço RPC, necessita-se certificar que o **rpcbind** está,
primeiramente, ativo e em execução.

Para distribuições linux que o systemd não cuidará automaticamente das intradependências
de serviço que existem entre rpcbind e ypserv, deverá ser executado os seguintes comandos
```
>> systemctl start rpcbind
```
Para as demais distribuições que contam com o systemd, o comando será
```
>> systemctl start ypserv
```
Para a confirmação que o serviço está ativo, pode-se executar o comando
```
>> rpcinfo -p | grep ypserv
```
O próximo passo será a configuração do Makefile, encontrado no diretório **/var/yp**. As
alterações a serem feitas serão
```
NOPUSH=true
MINUID=500
MINGID=500
MERGE_PASSWD=true
MERGE_GROUP=true
```
Para a inicialização do NIS server utilizando ypinit basta executar o seguinte comando,
conforme a arquitetura do computador
- Para arquiteturas 32-bits
```
>> /usr/lib/yp/ypinit -m
```
- Para arquiteturas 64-bits
```
>> /usr/lib64/yp/ypinit -m
```

### Instalando e configurando o agregador de tarefas SLURM
O SLURM será instalado através do comando a seguir
```
>> apt install slurm-wlm -y
```
Com o SLURM instalado, basta o configurar. Inicialmente iremos preparar o ambiente para
começar a configuração
```
>> cd /etc/slurm
>> cp /usr/share/doc/slurm-client/examples/slurm.conf.simple.gz .
>> gzip -d slurm.conf.simple.gz
>> mv slurm.conf.simple slurm.conf
```
O primeiro arquivo para edição será o **slurm.conf**, iremos editá-lo com o seguinte comando
```
>> vi /etc/slurm/slurm.conf
```
Deve-se editar algumas opções deste arquivo, para indicar qual Raspberry Pi será o controlador,
a seguinte linha deverá ser incluída
```
SlurmctldHost=server_name(IP_ADDR_OF_SERVER)
```
Para indicar como o SLURM alocará seus recursos, a linha a seguir deverá ser incrementada
```
SelectType=select/cons_res
SelectTypeParameters=CR_Core
```
É possível modificar o nome do cluster através da seguinte modificação
```
ClusterName=rpicluster
```
O servidor deverá reconhecer cada nó, para habilitar este reconhecimento basta editar a
linha a seguir
```
NodeName=rpicluster0X NodeAddr=IP_ADDR_OF_NODE0X CPUs=4 State=UNKNOWN
```
onde deverá repor **rpicluster0X**, **IP_ADDR_OF_NODE0X** para os nomes apropriados.

O próximo passo será configurar a partição que será um agrupamento lógico de nós que o
SLURM usará para executar cargas de trabalho de tarefas. Neste passo será adicionado os nós,
basta editar a seguinte informação
```
PartitionName=mycluster Nodes=rpicluster[0X-0Y] Default=YES MaxTime=INFINITE State=UP
```
editando os valores conforme o necessário.

É necessário configurar o SLURM para suportar o isolamento do kernel do cgroups, isso
permitirá que o SLURM saiba quais recursos do sistema estão disponíveis para as cargas de
trabalho de tarefas utilizarem. O arquivo a ser editado será **cgroup.conf** e para editá-lo será
necessário dar o comando abaixo
```
>> vi /etc/slurm/cgroup.conf
```
Basta adicionar as linhas abaixo ao arquivo, salvá-lo e fechá-lo.
```
CgroupMountpoint="/sys/fs/cgroup"
CgroupAutomount=yes
CgroupReleaseAgentDir="/etc/slurm-llnl/cgroup"
AllowedDevicesFile="/etc/slurm-llnl/cgroup_allowed_devices_file.conf"
ConstrainCores=no
TaskAffinity=no
ConstrainRAMSpace=yes
ConstrainSwapSpace=no
ConstrainDevices=no
AllowedRamSpace=100
AllowedSwapSpace=0
MaxRAMPercent=100
MaxSwapPercent=100
MinRAMSpace=30
```
Também se torna necessário informar ao SLURM quais dispositivos ele tem acesso. Isto é
feito editando o arquivo **cgroup_allowed_devices_file.conf** através do comando
```
>> vi /etc/slurm/cgroup_allowed_devices_file.conf
```
Adiciona-se à este arquivo as seguintes informações
```
/dev/null
/dev/urandom
/dev/zero
/dev/sda*
/dev/cpu/*/*
/dev/pts/*
/sharedfs*
```
Como o servidor se encontra configurado, se torna necessário compartilhar esta configuração
com os nós. Como o NFS já foi configurado inicialmente, basta utilizá-lo para enviar os arquivos
de configuração do servidor aos nós. Para isto basta dar os comandos abaixo
```
>> cp slurm.conf cgroup.conf cgroup_allowed_devices_file.conf /mnt/nfsshare
>> cp /etc/munge/munge.key /mnt/nfsshare
```
Com as configurações finalizadas, basta habilitar e ativar o **munge**, **slurmd** e **slurmctld**
através dos comandos a seguir.
```
>> systemctl enable munge
>> systemctl start munge

>> systemctl enable slurmd
>> systemctl start slurmd

>> systemctl enable slurmctld
>> systemctl start slurmctld
```
Vale ressaltar que talvez seja necessário reinicializar o servidor após todas alterações feitas.

### Criando usuário
Os usuários criados para o uso do cluster serão criados a partir do comando abaixo
```
>> adduser <user_name>
```
onde **<user_name>** será o nome de usuário escolhido.

Após criar o usuário no servidor, será necessário atualizar o banco de dados para que os nós
possam "enxergá-los", e para isto será necessário executar o seguinte comando
```
>> cd /var/yp
>> make
```
Com todos os nós "enxergando" os novos usuários adicionados, basta cadastrá-los em cada
nó. Um script foi desenvolvido para facilitar este processo e poderá ser visto a seguir
```
#!/bin/bash
ssh-keygen -t sra -b 4096
cd .ssh/
usr_new=$(whoami)
for i in $(seq 1 3)
do
	ssh-copy-id $usr_new@no0$i
done
```
onde algumas poucas entradas à mão ainda serão necessárias, e este script deverá ser executado
em **/home/<user\>**

Feito estes passos uma vez, as próximas entradas em cada nó será de forma direta e sem
necessidade de autenticação.

## Configurando os nós
Após a instalação do SO nos nós, os comandos deverão ser executados como **super usuário
(su)**. Para isso, será utilizado o seguinte comando
```
>> sudo su
```
Para garantir o funcionamento do cluster é necessário que a data e hora estejam equivalentes
entre todos os nós, tanto no master quanto nos slaves. Por isso, ao ligar o equipamento, deverá
ser digitado no terminal
```
>> date –set "DIA MES ANO HORA:MINUTO:SEGUNDO"
```
onde **MES** é a abreviado com três letras (e.g., JAN, FEV, MAR). Para conferir se as datas estão
equivalentes basta executar o comando
```
>> date
```

### Atualizando o SO
Antes de começar a configuração dos nós, o sistema operacional deve ser atualizado. Como
feito no servidor, será digitado no terminal
```
>> apt update
>> apt upgrade -y
```

### Configurando a placa de rede
Será determinado um IP fixo para cada nó, para isso alguns arquivos deverão ser alterados.
Primeiro, criar um arquivo **/etc/hostname**
```
noXX
```
onde **XX** será trocado pelo numero do no, no nosso caso, 01, 02 ou 03.

O IP fixo será setado ao se escrever no arquivo **/etc/netplan/50-cloud-init.yaml**
```
network:
	version:2
	renderer:networkd
	ethernets:
		eth0:
			dhcp4: true
			addresses: [10.0.1.X/24]
			routes:
				- to: default
				  via: 10.0.1.1
			nameservers:
				addresses: [8.8.8.8, 1.1.1.1]
```
onde **X** será o número relativo ao nó. Para o servidor será 5, para os nós 6, 7 e 8. Após editado
o arquivo, deveremos aplicar as mudanças feitas através do seguinte comando
```
>> netplan apply
```
Em seguida, no arquivo **/etc/hosts** iremos adicionar as linhas
```
10.0.1.5 	servidor.cluster 	servidor
10.0.1.6 	no01.cluster 		no01
10.0.1.7 	no02.cluster 		no02
10.0.1.8 	no03.cluster 		no03
```
É necessário, agora, escrever no arquivo **/etc/hosts.equiv** os nomes dos nós da rede para que
possa ser feita a equivalência em todas as máquinas. O arquivo em questão ficaria:
```
servidor no01 no02 no03
```

### Instalando e configurando o protocolo NFS
Com o SO atualizado, será feita a instalação e configuração do protocolo NFS para os
clientes. Desta forma os nós poderão acessar arquivos pela rede como se tratasse de arquivos
locais.

Para se instalar o NFS para o cliente será executado o comando
```
>> apt install nfs-common -y
```
A pasta compartilhada será montada no **/mnt**, assim
```
>> chown nobody.nogroup /mnt
>> chmod -R 777 /mnt
```
Para fazer com que a pasta seja montada automaticamente quando os nós são ligados, deve-se
alterar o arquivo **/etc/fstab**. Adiciona-se a seguinte linha
```
<MASTER_IP>:/mnt/nfsshare /mnt nfs defaults 0 0
<MASTER_IP>:/home /home nfs defaults 0 0
```
Agora basta digitar no terminal
```
>> mount -a
```

### Instalando e configurando o protocolo NIS
Para instalar o NIS execute no terminal
```
>> apt install nis portmap -y
```
O arquivo **/etc/nsswitch.conf** deverá ser alterado adicionando nis ao final das linhas
```
passwd: 	files nis
shadow: 	files nis
group: 		files nis
dns: 		files nis
```
No arquivo **/etc/yp.conf**, escreve-se
```
ypserver <MASTER_IP>
```
onde **<MASTER_IP>** é o IP do server.

### Instalando e configurando o agregador de tarefas SLURM
Para instalar o slurm vamos executar no terminal
```
>> apt install slurmd slurm-cliente -y
```
Após instalado o slurm, iremos copiar os arquivos de configuração do master que estão presentes
na pasta compartilhada montada em **/mnt**.
```
>> cp /mnt/munge.key /etc/munge/munge.key
>> cp /mnt/slurm.conf /etc/slurm/slurm.conf
>> cp /mnt/cgroup* /etc/slurm
```
Para habilitar e iniciar o Munge, executa-se no terminal
```
>> systemctl enable munge
>> systemctl start munge
```
Finalmente podemos habilitar e iniciar o SLURM Daemon
```
>> systemctl enable slurmd
>> systemctl start slurmd
```
