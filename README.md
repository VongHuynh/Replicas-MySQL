# Replicas-MySQL master-slave use image and handmade mysql 5.7

1. Cài đặt Mysql Server docker container 

Khởi tạo một docker network:
$ docker network create replicanet

Sử dụng các dòng lệnh dưới đây để khởi tạo 3 MySQL container:

$ docker run -d --name=master --net=replicanet --hostname=master -p 3308:3306 \
  -v $PWD/d0:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=1 \
  --log-bin='mysql-bin-1.log'

$ docker run -d --name=slave1 --net=replicanet --hostname=slave1 -p 3309:3306 \
  -v $PWD/d1:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=2

$ docker run -d --name=slave2 --net=replicanet --hostname=slave2 -p 3310:3306 \
  -v $PWD/d2:/var/lib/mysql -e MYSQL_ROOT_PASSWORD=mypass \
  mysql/mysql-server:5.7 \
  --server-id=3
  
 2. Configuring Master và Slave
 [Optional] Nếu như bạn muốn sử dụng cơ chế semisynchronous replication, thì hãy chạy câu lệnh bên dưới:
Việc cài đặt plugin semisynchronous nhầm mục đích giảm thiểu việc lag dữ liệu trong quá trình đồng bộ data giữa master và slave.

$ docker exec -it master mysql -uroot -pmypass \
  -e "INSTALL PLUGIN rpl_semi_sync_master SONAME 'semisync_master.so';" \
  -e "SET GLOBAL rpl_semi_sync_master_enabled = 1;" \
  -e "SET GLOBAL rpl_semi_sync_master_wait_for_slave_count = 2;" \
  -e "SHOW VARIABLES LIKE 'rpl_semi_sync%';"
  
 Khởi tạo ở master node một user replicate để các slave có thể access vào và lấy dữ liệu:
 $ docker exec -it master mysql -uroot -pmypass \
  -e "CREATE USER 'repl'@'%' IDENTIFIED BY 'slavepass';" \
  -e "GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';" \
  -e "SHOW MASTER STATUS;"
  
3. Cài đặt Slaves Node
 $ for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
    -e "INSTALL PLUGIN rpl_semi_sync_slave SONAME 'semisync_slave.so';" \
    -e "SET GLOBAL rpl_semi_sync_slave_enabled = 1;" \
    -e "SHOW VARIABLES LIKE 'rpl_semi_sync%';"
done

Chạy lệnh bên dưới để setting Slave Node:
 $ for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
    -e "CHANGE MASTER TO MASTER_HOST='master', MASTER_USER='repl', \
      MASTER_PASSWORD='slavepass', MASTER_LOG_FILE='mysql-bin-1.000003';"

  docker exec -it slave$N mysql -uroot -pmypass -e "START SLAVE;"
done

Kiểm tra trạng thái slave replication trên slave1 và slave2:

$ docker exec -it slave1 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"
$ docker exec -it slave2 mysql -uroot -pmypass -e "SHOW SLAVE STATUS\G"

3. Testing kết quả cài đặt
$ docker exec -it master mysql -uroot -pmypass -e "CREATE DATABASE TEST; SHOW DATABASES;"

Kiểm tra ở slave node đã được đồng bộ sang hay chưa:

for N in 1 2
  do docker exec -it slave$N mysql -uroot -pmypass \
  -e "SHOW VARIABLES WHERE Variable_name = 'hostname';" \
  -e "SHOW DATABASES;"
done

link resource guide: https://viblo.asia/p/database-replication-with-docker-mysql-images-rails-application-bWrZnxRw5xw

----------------------------------------------------------------
# Run Docker MySQL master-slave replication mysql 8.0 user compose mysql 8.0
To run this examples you will need to start containers with "docker-compose" and after starting setup replication. See commands inside ./build.sh.

TEST:

$ docker exec mysql_master sh -c "export MYSQL_PWD=111; mysql -u root mydb -e 'create table code(code int); insert into code values (100), (200)'"

$ docker exec mysql_slave sh -c "export MYSQL_PWD=111; mysql -u root mydb -e 'select * from code \G'"


