# Galera-Maxscale

Galera Replication
1.	Install MariaDB Server and Galera. Start MariaDB Service.
2.	Confirm Galera directory, it is usually `/usr/lib64/galera-4` or `/usr/lib64/galera`
3.	Add below parameters in `/etc/my.cnf.d/server.cnf` file under [galera] block:
`
wsrep_on=ON
wsrep_provider=/usr/lib64/galera-4/libgalera_smm.so
wsrep_cluster_name="my_galera_cluster"
wsrep_node_name="node1"
wsrep_node_address="192.168.249.139"
wsrep_cluster_address="gcomm://192.168.249.137,192.168.249.138,192.168.249.139"
binlog_format=ROW
wsrep_sst_method=rsync
default_storage_engine=InnoDB
innodb_autoinc_lock_mode=2
bind-address=192.168.249.139
`

5.	Run “systemctl stop mariadb” on all servers.
6.	Run “galera_new_cluster” on 1 server and “systemctl start mariadb” on other servers.
a.	If there is a issue then open /var/lib/mysql/grastate.dat and change “safe_to_bootstrap: 0” to “safe_to_bootstrap: 1”
b.	Also, there can be issue of members in /var/lib/mysql/gvwstate.dat, then remove all members and save file.
7.	Run this command to check if number of nodes in cluster are correct - mysql -u root -e "SHOW STATUS LIKE 'wsrep_cluster_size';"
8.	Verify by creating database in one server and check its existence from other servers.
9.	Also verify by stopping one server check "SHOW STATUS LIKE 'wsrep_cluster_size';" which will show updated cluster size and then start that server and check again.
10.	“SHOW STATUS LIKE 'wsrep_last_committed';” shows sequence number of node. Node with the highest value is considered the most up-to-date in terms of committed transactions.
