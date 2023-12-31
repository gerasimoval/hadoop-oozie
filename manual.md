# 1. Установка hadoop (single-node-cluster)

## 1.1. Скачивание hadoop архива
```
$ cd ~/Downloads
$ mkdir ozzie && cd oozie
$ wget https://dlcdn.apache.org/hadoop/common/hadoop-3.3.6/hadoop-3.3.6.tar.gz
$ tar -xzvf hadoop-3.3.6.tar.gz
$ sudo mv hadoop-3.3.6 /usr/local/hadoop
$ export PATH=$PATH:/usr/local/hadoop/bin
$ which hadoop
/usr/local/hadoop/bin/hadoop
```
## 1.2. Настройка hadoop
Указать $JAVA_HOME переменную окружения для Hadoop в /usr/local/hadoop/etc/hadoop/hadoop-env.sh file.
```
$ sudo vi /usr/local/hadoop/etc/hadoop/hadoop-env.sh
```
```
export JAVA_HOME=$(readlink -f $(which java) | sed "s:bin/java::")
```

Так же отредактировать ~/.bashrc файл:
```
$ vi ~/.bashrc
```
```
export JAVA_HOME=$(readlink -f $(which java) | sed "s:bin/java::")
export HADOOP_HOME=/usr/local/hadoop
export HADOOP_MAPRED_HOME=$HADOOP_HOME
export HADOOP_COMMON_HOME=$HADOOP_HOME
export HADOOP_HDFS_HOME=$HADOOP_HOME
export YARN_HOME=$HADOOP_HOME
export HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
export HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib"
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
```
```
$ source ~/.bashrc
$ hadoop version
Hadoop 3.3.6
...
```

Hadoop успешно установлен в standalone mode.

# 2. Настройка HDFS

## 2.1. Создание пользователя

Необходимо создать пользователя с sudo правами для запуска hadoop и oozie.

### Создание пользоваеля и группы
```
$ sudo groupadd testlab
$ sudo useradd –ingroup testlab testlab
$ passwd testlab
```

### Добавление sudo прав
```
$ sudo adduser testlab sudo
```
### Изменение владельца HADOOP_HOME на testlab
```
$ sudo chown -R testlab:testlab /usr/local/hadoop
```
### Создание SSH ключа
```
$ su testlab
$ ssh-keygen -t rsa -P ""
$ cat $HOME/.ssh/id_rsa.pub >> $HOME/.ssh/authorized_keys
$ ssh localhost   # Проверка входа без пароля
$ exit
```
## 2.2. Настройка namenode & datanode
```
$ sudo mkdir -p /app/hadoop/tmp
$ sudo chown testlab:testlab /app/hadoop/tmp
```
## 2.3. Натройка HDFS каталога и URI
```
$ sudo vi /usr/local/hadoop/etc/hadoop/core-site.xml
```
```
<configuration>
  ...
  <property>
      <name>hadoop.tmp.dir</name>
      <value>/app/hadoop/tmp</value>
  </property>
  <property>
    <name>fs.default.name</name>
    <value>hdfs://localhost:54310</value>
  </property>
  ...
</configuration>
```

## 2.4. Настройка Haddop tracker
```
$ cp /usr/local/hadoop/etc/hadoop/mapred-site.xml.template /usr/local/hadoop/etc/hadoop/mapred-site.xml
$ sudo vi /usr/local/hadoop/etc/hadoop/mapred-site.xml
```
```
<configuration>
  ...
  <property>
    <name>mapred.job.tracker</name>
    <value>localhost:54311</value>
</property>
  ...
</configuration>
```

## 2.5. Настройка namenode и datanode каталогов
```
$ sudo mkdir -p /usr/local/hadoop_store/hdfs/namenode
$ sudo mkdir -p /usr/local/hadoop_store/hdfs/datanode
$ sudo chown -R testlab:testlab /usr/local/hadoop_store
```
## 2.6. Настройка HDFS
```
$ sudo vi /usr/local/hadoop/etc/hadoop/hdfs-site.xml
```
```
<configuration>
  ...
  <property>
    <name>dfs.replication</name>
    <value>1</value>
  </property>
  <property>
    <name>dfs.namenode.name.dir</name>
    <value>file:/usr/local/hadoop_store/hdfs/namenode</value>
  </property>
  <property>
    <name>dfs.datanode.data.dir</name>
    <value>file:/usr/local/hadoop_store/hdfs/datanode</value>
  </property>
  ...
</configuration>
```

## 2.7. Старт HDFS
```
$ cd /usr/local/hadoop_store/hdfs/namenode/
$ hadoop namenode -format
$ cd /usr/local/hadoop/sbin 
$ ./start-all.sh
$ jps   # Проверка старта Hadoop cluster
```
    Namonode: http://localhost:50070
    Datanode: http://localhost:50090
    Resource manager: http://localhost:8088

# 3. Установка Ozzie
## 3.1. Установка maven
   
### Для сборки Oozie необходимо установть maven
```
$ dnf install maven
```

## 3.2. Скачивание Ozzie
```
$ cd ~/Downloads/ozzie
$ wget http://mirror.downloadvn.com/apache/oozie/5.2.1/oozie-5.2.1.tar.gz
$ tar -xzvf oozie-5.2.1.tar.gz
```
# Скачивание Ozzie расширения необходимого для web консоли
```
$ wget http://archive.cloudera.com/gplextras/misc/ext-2.2.zip
```
## 3.3. Сборка Oozie
Перед сборкой необходимо отредактировать репозитории, т.к проект мертвый адрес репозитория из исходников мертвый

Добавить в файл ~/oozie-5.2.1/pom.xml в секцию <repositories>
```
<repositories>
...
  <repository>
      <id>pentaho-public</id>
      <name>Pentaho Public</name>
          <url>https://repo.orl.eng.hitachivantara.com/artifactory/pnt-mvn/</url>
          <releases>
              <enabled>true</enabled>
              <updatePolicy>daily</updatePolicy>
          </releases>
          <snapshots>
              <enabled>true</enabled>
              <updatePolicy>interval:15</updatePolicy>
          </snapshots>
  </repository>
...
<repositories>
```

$ cd ~/oozie-5.2.1
$ ./bin/mkdistro.sh -Dmaven.test.skip=true -P hadoop-3 -Dhadoop.version=3.3.6

После завршения сборки, архив появится в distro/target/oozie-5.2.1-distro.tar.gz.

$ mkdir ../oozie-dist
$ tar -xzvf distro/target/oozie-5.2.1-distro.tar.gz -C ../oozie-dist
$ cd ../oozie-dist/oozie-5.2.1

## 3.4. Добалвение библиотек Oozie libext

Создать libext каталог в oozie-dist/oozie-5.2.1/ и распаковать Ext (скачанный в пнкте 3.1).

$ mkdir libext
$ mv ~/Downloads/ozzie/ext-2.2.zip libext/

### Так же необходимо скопировать hadoop jar'ы из hadoop шары в libext.

## 3.5. Старт Oozie демона

### Загрузить Oozie sharelib на HDFS
$ ./bin/oozie-setup.sh sharelib create -fs hdfs://localhost:54310
### Инициализация Oozie бд
$ ./bin/ooziedb.sh create -sqlfile oozie.sql -run
### Старт демона
$ ./bin/oozied.sh start
![oozie](oozie_started.jpg|width=500)
</br>
<img src="https://github.com/gerasimoval/hadoop-oozie/blob/main/oozie_started.jpg" width="500">
</br>
Web консоль Oozie должна быть доступна по адресу http://localhost:11000/oozie/
