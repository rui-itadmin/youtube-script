# EP4.1: Convert DNS slave to master
https://youtu.be/AImdJffolo0

## In this episode
---
- Fine-tune DNS master/slave settings
- Demonstrate DNS master failure
- Convert DNS slave to master
- Reinstall BIND9 and configure it as a slave server

## add notify for dns-master 
cat /etc/bind/named.conf.local  
cat /var/cache/bind/lab.local.zone  
dig myownlab.lab.local +short @192.168.122.100  
dig myownlab.lab.local +short @192.168.122.101  
dig myownlab.lab.local +short @192.168.122.102  

sudo vi /etc/bind/named.conf.local  

```
zone "lab.local" IN {
        type master;
        file "lab.local.zone";
        allow-transfer { 192.168.122.102; };
        also-notify { 192.168.122.102; };
};
```
sudo rndc reload  

## add text format for dns-slave
cat /var/cache/bind/lab.local.zone  

sudo vi /etc/bind/named.conf.local  
```
zone "lab.local" IN {
        type slave;
        file "lab.local.zone";
        masters { 192.168.122.101; };
        masterfile-format text;
};
```

sudo systemctl restart named  

cat /var/cache/bind/lab.local.zone  



## dns-master-101 add a new record and verify that everything is going fine.
sudo vi /var/cache/bind/lab.local.zone  
> update, serial; add a record  
sudo rndc reload  

## dns2 check new record
cat /var/cache/bind/lab.local.zone  



## Pretend that dns1 has crashed
sudo apt purge bind9  
sudo rm -rf /etc/bind/ /var/cache/bind/  


## Set dns2 as the master.
> take keepalived vip  
dig aaa.lab.local +short @192.168.122.100  
dig aaa.lab.local +short @192.168.122.101 #got timeout  
dig aaa.lab.local +short @192.168.122.102  

sudo vi /etc/bind/named.conf.local  

```before
zone "lab.local" IN {
        type slave;
        file "lab.local.zone";
        masters { 192.168.122.101; };
        masterfile-format text;
};
```

```after
zone "lab.local" IN {
        type master;
        file "lab.local.zone";
        allow-transfer { 192.168.122.101; };
        also-notify { 192.168.122.101; };
};
```
sudo rndc reload  


## Make dns-101 a slave
sudo apt update;sudo apt install bind9 -y  

sudo vi /etc/bind/named.conf.local  
```
zone "lab.local" IN {
        type slave;
        file "lab.local.zone";
        masters { 192.168.122.102; };
        masterfile-format text;
};
```

sudo vi /etc/bind/named.conf.options  
```
        listen-on-v6 { ::1; };
        #listen-on-v6 { any; };
        allow-query     { 192.168.122.0/24; localhost;};
        forwarders      { 8.8.8.8; 8.8.4.4; };
        listen-on port 53 { any; };
        listen-on port 953 { localhost; };
```
ls /var/cache/bind/ -al  
sudo systemctl restart named  
ls /var/cache/bind/ -al  
cat ls /var/cache/bind/lab.local.zone  
dig aaa.lab.local +short @192.168.122.101  


## dns-master-102 add a new record
sudo vi /var/cache/bind/lab.local.zone
> add bbb.lab.local for verity
sudo rndc reload  

## dns1-slave-101 check zone file
cat /var/cache/bind/lab.local.zone  
dig bbb.lab.local +short @192.168.122.102  


