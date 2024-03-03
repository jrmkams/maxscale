<h2><b>Mariadb Database Master-Master Replication With MaxScale Load Balancer</b> </h2>
<h3><b>Context :</b> We are working to make a replication system between two Mariadb databases and to balance the workload automaticaly between them using maxscale </h3>
<h4>Author : Jeremie Kamuina</h4>
<h2><b>Prerequisites </b></h2>
<ul>
  <li>2 Amazon Linux 2023 instances</li>
  <li>1 Ubuntu instance</li>
  <li>A SSH client (Mobaxterm or Putty)</li>
  <li>Allow traffic on 3306 port for Mysql/Aurora, SSH on 22 and on both 4006 and 4008 for current TCP in the inbound rules of the instances security group</li>
  <li>Microsoft Workbench</li>
  <li>Minimal knowledge of linux command line</li>
</ul>
<h2>Steps</h2>
<ol>
  <li>Install MariaDb and phpMyAdmin (LAMP) on both Amazon Linux 2023 instances</li>
  <cite>Follow the instructions on : <a href="https://docs.aws.amazon.com/linux/al2023/ug/ec2-lamp-amazon-linux-2023.html">https://docs.aws.amazon.com/linux/al2023/ug/ec2-lamp-amazon-linux-2023.html</a></cite>
  <p>Restart mariadb : <code>sudo systemctl restart mariadb</code></p>
  <li>Configure Master Replication on both Mariadb Servers</li>
  <ol type="i">
    <li >
      SSH the two MariaDB Servers
    </li>
    <li >
     open the first server mariaDb configuration file typing <code>sudo nano /etc/my.cnf.d/mariadb-server.cnf</code>
    </li>
    <li>paste the following code bellow [mariadb-10.5] section</li>
    <code>log-bin
server_id=1
log-basename=master1
binlog-format=mixed </code>
     <li >
     open the second server mariaDb configuration file typing <code>sudo nano /etc/my.cnf.d/mariadb-server.cnf</code>
    </li>
    <li>paste the following code bellow [mariadb-10.5] section</li>
    <code>log-bin
server_id=2
log-basename=master2
binlog-format=mixed </code>
   <br> <strong>NB: Do the following steps on both mariaDb servers</strong>
    <li>Restart mariaDb typing <code>sudo systemctl restart mariadb</code></li>
    <li>type <code>mariadb -u root -p</code> to connect to mariadb with the root user</li>
    <li>type your root user password</li>
    <li>type <code>SHOW MASTER STATUS;</code> to display the positioning of the next event in the binary logging file </li>
  </ol>
  <li>Create one replication user in both mariaDB databases typing on each server</li>
  <code>CREATE USER 'nom_utilisateur'@'adresse_IP' IDENTIFIED BY 'mot_de_passe';</code>
  <Grand the needed privileges
  <code>GRANT REPLICATION SLAVE ON *.* TO 'nom_utilisateur'@'adresse_IP';
    FLUSH PRIVILEGES;</code>
    <li>Configure slave replication</li>
    type <code>STOP slave;</code>
    In the first server type <code> CHANGE MASTER TO
MASTER_HOST = server2_ip_address,
MASTER_USER = 'replication_user2',
MASTER_PASSWORD = 'replication_user2_password',
MASTER_LOG_FILE='MASTER2_LOG_FILE', 
MASTER_LOG_POS=MASTER2_POSITION;
</code><br>
    In the second server type <code> CHANGE MASTER TO
MASTER_HOST = server1_ip_address,
MASTER_USER = 'replication_user1',
MASTER_PASSWORD = 'replication_user1_password',
MASTER_LOG_FILE='MASTER1_LOG_FILE', 
MASTER_LOG_POS=MASTER1_POSITION;
</code><br>
type <code>START SLAVE;</code> on both servers<br>
check  the slave replication activity on the both servers typing <code>SHOW SLAVE STATUS \G</code><br>
Review the output and make sure there is no error and the Slave_IO_Running and Slave_SQL_Running  are set to  yes
    <li>Create a database in of of the mariadb servers</li>
    type <code>CREATE DATABASE database_name;</code>
    <br>check on the second Mariadb server if the created database was replicated
    <br> type <code>SHOW DATABASES;</code><br>
    <i>If you find the created databases in both MariaDB servers congratulations, your replication worked successfully</i>
    <li>Make Load Balacing with Maxcale</li>
    <ol>
      <li>Install maxscale on your Ubuntu instance</li>
      <p>Type the following codes</p>
      <code>curl -LsS https://r.mariadb.com/downloads/mariadb_repo_setup | sudo bash
        sudo nano /etc/apt/sources.list.d/mariadb.list
        deb [arch=amd64] http://downloads.mariadb.com/MaxScale/latest/ubuntu focal main
        sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
        sudo apt-key adv --fetch-keys 'https://mariadb.org/mariadb_release_signing_key.asc'
        sudo apt-get update
        sudo apt-get install maxscale</code><br>
      <li>Start and enable MaxScale Service</li>
      <code>sudo systemctl start maxscale
        sudo systemctl enable maxscale</code>
      <li>Create a maxcale user in both MariaDB databases</li>
      <code>CREATE USER 'maxscale_user'@'%' IDENTIFIED BY 'maxscale_password';
            GRANT ALL PRIVILEGES ON *.* TO 'maxscale_user'@'%';
            FLUSH PRIVILEGES;</code>
      <li>Configure maxscale</li>
      <p>Open the configuration file type</p> <code>sudo nano /etc/maxscale/maxscale.cnf
</code>
      <p>
        in Server definitions section paste the following text
      </p>
      <code>[server1]
type           = server
address        = ip_serveur_mariadb1
port           = 3306
protocol       = MariaDBBackend

[server2]
type           = server
address        = ip_serveur_mariadb2
port           = 3306
protocol       = MariaDBBackend</code>

<p>in Monitor for the servers section paste the following text</p>
<code>type=monitor
module=mariadbmon
servers=server1,server2
user=maxscale_user
password=maxscale_password
monitor_interval=2000</code>
<p>in Service definitions section paste the following text</p>
<code>[Read-Only-Service]
type=service
router=readconnroute
servers=server1,server2
user=maxscale_user
password=maxscale_password
router_options=slave</code><br><br>
<code>[Read-Write-Service]
type=service
router=readwritesplit
servers=server1,server2
user=maxscale_user
password=maxscale_password</code>
<li>Restart maxscale</li>
type <code>sudo systemctl restart maxscale</code>
<li>Try to connect with MySql workbench</li>
<p>type your Ubuntu ip address as the hostname, youy maxcale_user username as the username and his password you created in preceding steps</p>
<p>Try to create a table </p>
<p>Review the both mariadb server to have the created table in both of them</p>
<p>Congratulations you did it</p>
<li>check if the connection worked</li>
type <code>maxctrl list servers</code> in your ubuntu server
    </ol>
</ol>

<address><a href ="mailto:jrmkams@gmail.com">mail : jrmkams@gmail.com</a></address>
