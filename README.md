# Galera-Maxscale

## Galera Replication
1.	Install MariaDB Server and Galera on 3 servers. Start MariaDB Service.
2.	Confirm Galera directory, it is usually `/usr/lib64/galera-4` or `/usr/lib64/galera`
3.	Add below parameters in `/etc/my.cnf.d/server.cnf` file under [galera] block:
```
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so
wsrep_cluster_name="my_galera_cluster"
wsrep_node_name="node1"
wsrep_node_address="x.x.x.1x(or hostname)"
wsrep_cluster_address="gcomm://x.x.x.1x(or hostname),x.x.x.2x(or hostname),x.x.x.3x(or hostname)"
binlog_format=ROW
wsrep_sst_method=rsync
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
bind-address=192.168.249.139
```

5.	Run `systemctl stop mariadb` on all servers.
6.	Run `galera_new_cluster` on master server and `systemctl start mariadb` on other servers.
a.	If there is an issue during cluster start then open `/var/lib/mysql/grastate.dat` and change `safe_to_bootstrap: 0` to `safe_to_bootstrap: 1`
b.	Also, there can be issue of members in `/var/lib/mysql/gvwstate.dat` then remove all members and save file.
7.	Run this command to check if number of nodes in cluster are correct `mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"`
8.	Verify by creating database in one server and check its existence from other servers.
9.	Also verify by stopping one server and then check `mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"` which will show updated cluster size and then start that server and check again.
10.	`mysql -u root -e "SHOW STATUS LIKE 'wsrep_last_committed';"` shows sequence number of node. Node with the highest value is considered the most up-to-date in terms of committed transactions.







## Maxscale High-Availibilty
1.	Download Maxscale from https://mariadb.com/downloads/community/maxscale/ and ftp to servers where you will run maxscale.
2.	Install by method given below
a.	`rpm -ivh libatomic-4.8.5-44.el7.x86_64.rpm`
b.	`rpm -ivh libicu-50.2-4.el7_7.x86_64.rpm`
c.	`rpm -ivh libtool-ltdl-2.4.2-22.el7_3.x86_64.rpm`
d.	`rpm -ivh unixODBC-2.3.1-14.el7.x86_64.rpm`
e.	`rpm -ivh maxscale-23.08.4-1.rhel.7.x86_64.rpm`
3.	Install MariaDB client on your Maxscale Server.
4.	Make sure `/var/run/maxscale` and `/var/lib/maxscale/maxscale.cnf.d` exists
5.	If you are facing “Executable path is not absolute” error then edit service file `/usr/lib/systemd/system/maxscale.service` and check `ExecStartPre` if there is + sign then remove it then run `systemctl daemon-reload` and try to start service.
6.	If maxscale service is enabled and service is unable to start at boot due to “/var/run/maxscale” not present then create a bash script to make directory and change its user and group to “maxscale” and make it executable, then add it in service file `/usr/lib/systemd/system/maxscale.service` in [Service] block as:
```
[Service]
Type=forking
Restart=on-abort
RuntimeDirectory=maxscale
ExecStartPre=/bin/bash /usr/lib/systemd/system/maxscale_start.sh
ExecStartPre=/usr/bin/install -d /var/run/maxscale -o maxscale -g maxscale
PIDFile=/var/run/maxscale/maxscale.pid
User=maxscale
Group=maxscale
ExecStart=/usr/bin/maxscale
```
7.	If you are facing `/usr/bin/install: cannot change owner and permissions of ‘/var/run/maxscale’: Operation not permitted` then run `chown maxscale:maxscale /var/run/maxscale` and `chmod 755 /var/run/maxscale` then run `systemctl daemon-reload`
8.	Start service by `systemctl start maxscale`
9.	Create a user and give privlages on DB to maxscale server as:
```
CREATE USER 'maxscale'@'maxscale_server_ip' IDENTIFIED BY 'abc';
GRANT SELECT ON mysql.user TO 'maxscale'@'maxscale_server_ip';
GRANT SELECT ON mysql.db TO 'maxscale'@'maxscale_server_ip';
GRANT SELECT ON mysql.tables_priv TO 'maxscale'@'maxscale_server_ip';
GRANT SELECT ON mysql.roles_mapping TO 'maxscale'@'maxscale_server_ip';
GRANT SLAVE MONITOR ON *.* TO 'maxscale'@'maxscale_server_ip' IDENTIFIED BY 'abc';
GRANT SHOW DATABASES ON *.* TO 'maxscale'@'maxscale_server_ip';
GRANT REPLICATION CLIENT on *.* to 'maxscale'@'maxscale_server_ip';
GRANT SELECT ON mysql.* TO 'maxscale'@'maxscale_server_ip';
FLUSH PRIVILEGES;
```
10.	Edit `/etc/maxscale.cnf` as below:
```
[maxscale]
threads=auto
 
[node1]
type=server
address=x.x.x.4x(or hostname)
port=3306
protocol=MariaDBBackend
 
[node2]
type=server
address=x.x.x.5x(or hostname)
port=3306
protocol=MariaDBBackend
 
[node3]
type=server
address=x.x.x.6x(or hostname)
port=3306
protocol=MariaDBBackend
 
[Galera-Monitor]
type=monitor
module=galeramon
servers=node1,node2,node3
user=maxscale
password=abc
monitor_interval=2s
 
[Read-Only-Service]
type=service
router=readconnroute
servers=node2,node3
user=maxscale
password=abc
router_options=slave
 
[Read-Write-Service]
type=service
router=readwritesplit
servers=node1
user=maxscale
password=abc
 
[Read-Only-Listener]
type=listener
service=Read-Only-Service
protocol=mariadbprotocol
port=4008
 
[Read-Write-Listener]
type=listener
service=Read-Write-Service
protocol=mariadbprotocol
port=4006
```
11.	Check maxscale logs by `tail -100 /var/log/maxscale/maxscale.log`
12.	Check `SHOW VARIABLES LIKE 'event_scheduler';` it should be “ON” if not then run `SET GLOBAL event_scheduler = ON;`
13.	Verify using `maxctrl list services` and `maxctrl list servers`
14.	Connect to maxscale server from one DB server using `mysql -u'maxscale' -h'x.x.x.4x' -P'4008' -p” and check “SHOW PROCESSLIST;`
15.	Test by creating database from maxscale server and check if it is visible in DB server.







## Maxscale as Load Balancer (with Keepalived)
1.	Setup a Galera Cluster with 3 nodes
2.	Setup Maxscale on 2 servers and make sure both are connected to Galera cluster.
3.	Install keepalived on both maxscale servers.
4.	Edit `/etc/keepalived/keepalived.conf` on both servers as:
```
! Configuration File for keepalived
 
vrrp_script check_maxscale {
    script "/bin/systemctl status maxscale"
interval 2
}
 
vrrp_instance VI_1 {
    state MASTER
    interface ens33
    virtual_router_id 51
    priority 11
    advert_int 1
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        x.x.x.x(virtual IP)
    }
    track_script {
    check_maxscale
    }
 
}
```
5.	Edit virtual_ipaddress(same on both maxscale servers), interface, state as `BACKUP` or `MASTER` as per requirement and start keepalived service on master and then start on slave.
6.	Check on master by `ifconfig` or `ip a` after starting service. It will display a virtual IP in active interface section.
7.	Test failover by shutting down master server and check on backup server, it should display same virtual IP in active interface section.
