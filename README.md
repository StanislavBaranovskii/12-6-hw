# Домашнее задание к занятию 12.6. «`Репликация и масштабирование. Часть 1`» - `Барановский Станислав`

### Инструкция по выполнению домашнего задания

1. Сделайте fork [репозитория c шаблоном решения](https://github.com/netology-code/sys-pattern-homework) к себе в Github и переименуйте его по названию или номеру занятия, например, https://github.com/имя-вашего-репозитория/gitlab-hw или https://github.com/имя-вашего-репозитория/8-03-hw).
2. Выполните клонирование этого репозитория к себе на ПК с помощью команды `git clone`.
3. Выполните домашнее задание и заполните у себя локально этот файл README.md:
   - впишите вверху название занятия и ваши фамилию и имя;
   - в каждом задании добавьте решение в требуемом виде: текст/код/скриншоты/ссылка;
   - для корректного добавления скриншотов воспользуйтесь инструкцией [«Как вставить скриншот в шаблон с решением»](https://github.com/netology-code/sys-pattern-homework/blob/main/screen-instruction.md);
   - при оформлении используйте возможности языка разметки md. Коротко об этом можно посмотреть в [инструкции по MarkDown](https://github.com/netology-code/sys-pattern-homework/blob/main/md-instruction.md).
4. После завершения работы над домашним заданием сделайте коммит (`git commit -m "comment"`) и отправьте его на Github (`git push origin`).
5. Для проверки домашнего задания преподавателем в личном кабинете прикрепите и отправьте ссылку на решение в виде md-файла в вашем Github.
6. Любые вопросы задавайте в чате учебной группы и/или в разделе «Вопросы по заданию» в личном кабинете.

Желаем успехов в выполнении домашнего задания.

---

## Задание 1

На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.

*Ответить в свободной форме.*

---

## Задание 2

Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

### Установка docker

<details><summary>Установка docker на debian</summary>

```bash
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg
echo  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli docker-buildx-plugin docker-compose-plugin
docker --version
sudo systemctl status docker
sudo docker run hello-world
```
</details>

### Установка контейнеров mysql-серверов
```bash
#Master :
#sudo docker run -d --name replication-master -e MYSQL_ALLOW_EMPTY_PASSWORD=true -v ~/path/to/world/dump:/docker-entrypoint-initdb.d mysql:8.0
sudo docker run -d --name replication-master -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.0

#Slave :
sudo docker run -d --name replication-slave -e MYSQL_ALLOW_EMPTY_PASSWORD=true mysql:8.0

#Ifrastructure :
sudo docker network create replication
sudo docker network connect replication replication-master
sudo docker network connect replication replication-slave

sudo docker ps -a
sudo docker network ls
```
### Настройка mysql-серверов
```bash
#Master :
sudo docker exec replication-master apt update && sudo docker exec replication-master apt install -y nano
sudo docker exec -it replication-master mysql
  CREATE USER 'replication'@'%';
  GRANT REPLICATION SLAVE ON *.* TO 'replication'@'%';
  exit

sudo docker exec -it replication-master bash
  nano /etc/mysql/my.cnf  #

sudo docker cp replication-master:/etc/mysql/my.cnf /tmp/my_master.cnf

sudo docker restart replication-master
sudo docker exec -it replication-master mysql
  SHOW MASTER STATUS;
  FLUSH TABLES WITH READ LOCK;
  exit

sudo docker exec replication-master mysqldump world > /tmp/world.sql
sudo docker exec -it replication-master mysql
  SHOW MASTER STATUS;
  UNLOCK TABLES;
  exit

#Slave :
sudo docker exec replication-slave apt update && sudo docker exec replication-slave apt install -y nano
sudo docker cp /tmp/world.sql replication-slave:/tmp/world.sql
sudo docker exec -it replication-slave mysql
  CREATE DATABASE `world`;
  exit

sudo docker exec -it replication-slave bash
  mysql world < /tmp/world.sql

sudo docker exec -it replication-slave bash
  nano /etc/mysql/my.cnf

sudo docker cp replication-slave:/etc/mysql/my.cnf /tmp/my_slave.cnf

sudo docker restart replication-slave
sudo docker exec -it replication-slave mysql
  CHANGE MASTER TO MASTER_HOST='replication-master',
  MASTER_USER='replication', MASTER_LOG_FILE='mysql-bin.000001',
  MASTER_LOG_POS=156;
  START SLAVE;
  SHOW SLAVE STATUS\G
  exit

```
### Тестирование репликации

<details><summary>Тестирование репликации. БД sakila-db</summary>

```bash
#Master :
docker exec -it replication-master mysql
  USE world;
  INSERT INTO city (Name, CountryCode, District, Population) VALUES
  ('Test-Replication', 'ALB', 'Test', 42);
  exit

#Slave :
docker exec -it replication-slave mysql
  USE world;
  SELECT * FROM city ORDER BY ID DESC LIMIT 10;
  exit
```
</details>



![Скриншот состояния и режимы работы серверов](https://github.com/StanislavBaranovskii/12-6-hw/blob/main/img/12-6-2.png "Скриншот состояния и режимы работы серверов")

---

## Задание 3* 

Выполните конфигурацию master-master репликации. Произведите проверку.

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

---
