# In-Memory (outubro 2024) 📊

## Índice
1. [Considerações iniciais e pré-requisitos](#1-considerações-iniciais-e-pré-requisitos)
   - [Recursos usados](#recursos-usados)
   - [Tópicos não cobertos](#tópicos-não-cobertos)
2. [Provisionamento de recursos](#2-provisionamento-de-recursos)
   - [Criação de VCN (Virtual Cloud Network)](#21-criação-de-vcn-virtual-cloud-network)
   - [Criação de chave SSH](#22-criação-de-chave-ssh)
   - [Criação do Banco de Dados Oracle Database Service](#23-criação-do-banco-de-dados-oracle-database-service)
   - [Acessando o banco de dados pela primeira vez](#24-acessando-o-banco-de-dados-pela-primeira-vez)
   - [Configurando o ambiente](#25-configurando-o-ambiente)
3. [Testando a funcionalidade In-Memory](#3-testando-a-funcionalidade-in-memory)
   - [Habilitando in-memory no banco](#31-habilitando-in-memory-no-banco)
4. [Finalizando a sessão](#4-finalizando-a-sessão)
   - [Desligando banco de dados](#41-desligando-banco-de-dados)
   - [Destruindo o banco de dados](#42-destruindo-o-banco-de-dados)

## 1. Considerações iniciais e pré-requisitos

### Recursos usados:

#### OCI (all free tier)
- Armazenamento Oracle Object Storage
- Virtual Cloud Network
- Oracle Database Cloud Service (Virtual Machine – 30 dias gratuito)

#### Local (opcionais)
- Editor de texto

### Tópicos não cobertos:

#### Instalação dos softwares na máquina host:
- Oracle SQL Developer
- Gerador de chaves SSH
- Cliente de SSH

#### Configuração:
- Conta gratuita no Oracle Cloud

## 2. Provisionamento de recursos

### 2.1. Criação de VCN (Virtual Cloud Network) 🌐

1. Entre na [OCI](https://www.oracle.com/cloud/sign-in.html)
2. Navegue no Menu e vá até **Rede -> Visão Geral -> Criar soluções de rede -> iniciar assistente de VCN**
3. Clique no botão **Iniciar Assistente de VCN**, escolha a opção **VCN com Conectividade de Internet** e **Iniciar Assistente de VCN**
4. Dê um nome para a sua VCN, escolha o compartimento para criação e os blocos CIDR. Se preferir, mantenha as sugestões. Clique em **Próximo**
5. Confira as configurações e clique em **PRÓXIMO**
6. Confira novamente e clique em **Criar**

Em alguns segundos a sua rede estará provisionada. Clique no botão **Exibir Rede Virtual na Nuvem**.

Para que seja possível acessar o banco de dados externamente, como por exemplo através do DBeaver instalado localmente em sua máquina, será necessário abrir a porta do Listener (1521). Na lista de recursos à esquerda escolha **Lista de Segurança**.

#### Escolha a sub-rede PÚBLICA:

1. Vamos editar a SL (Security List) pública, clique em **Default Security List for <nome_da_vcn>**
2. Clique no botão **Adicionar Regras de Entrada**
3. Abra para todos os endereços IP (CIDR DE ORIGEM: 0.0.0.0/0), mantenha o protocolo TCP e adicione a porta de destino 1521. Clique em **Adicionar Regras de Entrada**
4. A nova regra de firewall é adicionada à SL

### 2.2. Criação de chave SSH 🔑

1. Clique no botão do **Cloud Shell (CLI)** no canto superior direito do console do OCI
2. Execute os comandos abaixo para criar a sua chave. Escolha um nome fácil para lembrar depois:

```sh
mkdir .ssh
cd .ssh
ssh-keygen -b 2048 -t rsa -f chave1
```

> Obs.: durante a criação será solicitado uma senha. Ela é opcional e você pode prosseguir teclando [Enter].

3. Verifique o conteúdo de suas chaves através do comando `ls`
4. Copie e cole o conteúdo da chave pública e privada, ela será necessária durante a criação do banco de dados:

```sh
cat chave1.pub
```

### 2.3. Criação do Banco de Dados Oracle Database Service 🗄️

1. Com a rede pronta, vamos provisionar o banco de dados. Clique no menu, vá para **Oracle Database -> Oracle Base Database Service**
2. Defina o nome do SGBD conforme tela abaixo:
3. Escolha **Logical Volume Manager**
4. Escolha versão **Enterprise Edition Extreme Performance**
5. Adicione a chave1.pub que foi salva localmente em passo anterior
6. Escolha **licença incluída**
7. Escolha a rede criada nos passos anteriores e escolha a sub-rede pública
8. Desmarque as opções de diagnóstico, monitoramento e backup, e clique em **Próximo**
9. Defina a senha do usuário SYS
10. Desmarque o ativar backup e clique em **Criar sistema de banco de dados**

Aguarde o banco ser criado, pode levar mais de uma hora...

### 2.4. Acessando o banco de dados pela primeira vez 🛠️

Uma vez finalizado o provisionamento do banco de dados iremos configurar o ambiente carregando as tabelas que serão utilizadas. Para este acesso precisamos do endereço IP do servidor. Vá em **Oracle Database -> Oracle Base Database Service**.

1. Na lista de Sistemas de BD clique no nome que você escolheu anteriormente. Se não aparecer o seu banco de dados, confirme que o compartimento correto foi selecionado.
2. Na janela que se abre clique em **Nós** (na lista Recursos, abaixo do status do banco de dados) e copie o endereço IP do servidor.
3. Abra o **Cloud Shell** (botão à direita na barra superior do OCI) onde iremos acessar o ambiente recém-criado via SSH.

> Obs.: No primeiro acesso será necessário confirmar a autenticidade do servidor, entre com o valor `yes` quando solicitado.

```sh
cd .ssh
ssh -i ./chave1 opc@<seu_ip>
sudo su - oracle
sqlplus / as sysdba
```

Siga os passos acima para confirmar que a conectividade ao banco de dados está funcionando conforme esperado. Na sequência pode sair do SQL*Plus com o comando `exit`.

### 2.5. Configurando o ambiente ⚙️

Agora você copiará os scripts de banco de dados para criação e carga das tabelas que utilizaremos em nossos exemplos:

```sh
cd /tmp
mkdir tdc
cd tdc
wget https://objectstorage.sa-saopaulo-1.oraclecloud.com/n/groyzeiavcrz/b/Labs_ABD/o/Loader.zip
unzip Loader.zip
cd Loader
chmod 775 setup.sh
nohup ./setup.sh &
```

Os scripts serão executados em background, deve demorar uns 10 a 20 minutos. Pode ir pegar um café.

Acesse o SQL*Plus para conferir se a carga terminou. Se a sua sessão tiver caído desde a etapa anterior, execute os 4 passos abaixo, caso contrário, apenas o último comando:

```sh
cd .ssh
ssh -i ./chave1 opc@<seu_ip>
sudo su - oracle
sqlplus c##tdc/TDC#inov_
```

Uma boa maneira de verificar se a carga acabou é validando a tabela `TBL_MOVIES_GENRES`, pois é a última a ser carregada:

```sql
SELECT COUNT(*) FROM tbl_movies_genres;
```

## 3. Testando a funcionalidade In-Memory 🧪

A partir de agora estamos prontos para analisar os planos de execução de queries de diferentes tipos e complexidades. A sequência abaixo tem um conjunto de consultas que serão executadas uma primeira vez com o InMemory desabilitado e depois novamente com o InMemory habilitado.

Para efeito de comparação, anote os custos de execução de cada query na primeira rodada e depois anote novamente, ao reexecutar as consultas com o InMemory habilitado.

```sql
-- Aparições do personagem Darth Vader
explain plan for
SELECT DISTINCT character_name, actor_id
FROM tbl_characters
WHERE character_name like 'Darth Vader%';
SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'basic +predicate +cost'));
```

Caso esteja curioso em saber o resultado desta e das demais consultas, você pode executá-la sem o `explain plan`. Recomendo também adicionar ao final a cláusula `ROWNUM <= 10` para evitar que muitas linhas sejam carregadas na tela.

```sql
-- Aparições do personagem Darth Vader
SELECT DISTINCT character_name, actor_id
FROM tbl_characters
WHERE character_name like 'Darth Vader%'
AND rownum <= 10;
```

Execute as consultas abaixo sempre anotando o custo de execução no armazenamento em linha sem o uso do InMemory.

```sql
-- Atores que interpretaram Darth Vader
explain plan for
SELECT DISTINCT a.actor_id, a.actor_name
FROM tbl_characters c, tbl_actors a
WHERE c.actor_id = a.actor_id
AND c.character_name like 'Darth Vader%';
SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'basic +predicate +cost'));
```

```sql
-- Quantos personagens cada um desses atores interpretou
explain plan for
SELECT a.actor_id, a.actor_name, count(*) qtde_personagens
FROM tbl_characters c, tbl_actors a
WHERE c.actor_id = a.actor_id
AND a.actor_id in (15152, 21132, 25451, 1728954, 1728453, 24342)
GROUP BY a.actor_id, a.actor_name
ORDER BY qtde_personagens DESC;
SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'basic +predicate +cost'));
```

Perceba que agora iniciaremos queries com características analíticas, consultando diversas tabelas, com maior número de linhas e agregações. Consequentemente o custo associado também será maior.

```sql
-- Quais os filmes que o intérprete do Darth Vader que mais fez personagens já fez?
explain plan for
SELECT mm.original_title, c.character_name, round(to_number(mm.popularity, '999D9999999999', 'NLS_NUMERIC_CHARACTERS = ''.'''), 2) popularidade, to_date(release_date, 'dd/mm/yy') data_lancamento
FROM tbl_characters c, tbl_movies_characters mc, stg_movies_metadata mm
WHERE c.character_id = mc.character_id
AND mc.movie_id = mm.id
AND c.actor_id = 15152
ORDER BY popularidade DESC;
SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'basic +predicate +cost'));
```

A consulta abaixo adiciona ainda mais complexidade, usando a maior tabela do banco, `STG_RATINGS`, que tem 26.024.289 registros.

```sql
-- Qual a nota média para os filmes interpretados por James Earl Jones
explain plan for
SELECT mm.original_title, c.character_name, round(to_number(mm.popularity, '999D9999999999', 'NLS_NUMERIC_CHARACTERS = ''.'''), 2) popularidade, to_date(release_date, 'dd/mm/yy') data_lancamento, round(avg(to_number(r.rating, '999D99', 'NLS_NUMERIC_CHARACTERS = ''.''')), 2) nota_media
FROM tbl_characters c, tbl_movies_characters mc, stg_movies_metadata mm, stg_ratings r
WHERE c.character_id = mc.character_id
AND mc.movie_id = mm.id
AND c.actor_id = 15152
AND mm.id = r.movieid
GROUP BY mm.original_title, c.character_name, round(to_number(mm.popularity, '999D9999999999', 'NLS_NUMERIC_CHARACTERS = ''.'''), 2), to_date(release_date, 'dd/mm/yy')
ORDER BY popularidade DESC;
SELECT plan_table_output
FROM TABLE(DBMS_XPLAN.DISPLAY('plan_table',null,'basic +predicate +cost'));
```

Consequentemente o custo também é bastante alto. Note que também há diversas indicações `TABLE ACCESS FULL`, o que coopera substancialmente para o custo. Tipicamente a solução para minimizar esse custo seria a criação de índices, entretanto quanto mais índices nas tabelas, pior a performance nas operações transacionais.

### 3.1. Habilitando in-memory no banco 🧠

Agora vamos habilitar a funcionalidade InMemory do banco de dados. Este não é um comando dinâmico, portanto será necessário reiniciar a instância. Vamos configurar a InMemory área para usar 10GB.

```sql
alter system set inmemory_size=5G scope=spfile;
conn / as sysdba
shutdown immediate;
startup open;
conn c##tdc/TDC#inov_2021
```

Vamos colocar as tabelas individualmente em memória com prioridade HIGH. Dessa maneira elas começam a ser atribuídas imediatamente à InMemory Area. Se mantivermos a opção padrão a tabela só será alocada em memória quando a primeira consulta for realizada.

```sql
ALTER TABLE tbl_characters INMEMORY PRIORITY HIGH;
ALTER TABLE tbl_actors INMEMORY PRIORITY HIGH;
ALTER TABLE tbl_movies_characters INMEMORY PRIORITY HIGH;
ALTER TABLE stg_movies_metadata INMEMORY PRIORITY HIGH;
ALTER TABLE stg_ratings INMEMORY PRIORITY HIGH;
```

Podemos acompanhar a alocação em memória pela query abaixo. Note que levará alguns segundos até tudo subir. Nessa consulta também é possível observar o tamanho ocupado em disco pela tabela e o quanto ela ocupa em memória.

```sql
select owner, segment_name, bytes/1024/1024 tam_original_mb, inmemory_size/1024/1024 tam_memoria_mb, round(bytes / inmemory_size, 2) taxa_comp, round((inmemory_size / bytes)*100, 2) pct_orig
from v$im_segments;
```

Agora execute novamente todas as queries acima e anote os novos custos de execução. Você terá algo parecido com o apresentado abaixo:

A relação de custo antes e depois de ativar o in-memory foi de XX%.

## 4. Finalizando a sessão 🛑

### 4.1. Desligando banco de dados

Confirme a exclusão do banco de dados entrando com o nome do sistema e clique em **Encerrar Sistema de BD**. O banco de dados será encerrado definitivamente e não consumirá mais recursos de seu tenant.


