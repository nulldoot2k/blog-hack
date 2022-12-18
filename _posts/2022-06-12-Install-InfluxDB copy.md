---
# layout: post
# title: Install InfluxDB
# published: true
permalink: /influxdb.html
---

## Install InfluxDB

Step1

> **sudo curl -sL https://repos.influxdata.com/influxdb.key ``|`` sudo apt-key add -**

Step 2

> **sudo echo "deb https://repos.influxdata.com/ubuntu bionic stable" `|` sudo tee /etc/apt/sources.list.d/influxdb.list**

Step 3

> **sudo apt update**

Step 4

> sudo apt install influxdb

Step 5 

> sudo apt autoremove -y

Step 6

sudo systemctl status/start/restart/stop influxdb

sudo systemctl enable --now influxdb


## Configuring InfluxDB

> sudo vi /etc/influxdb/influxdb.conf

```bash
[http]
  enabled = true
  bind-address = "localhost:8086"
  auth-enabled = false
```

> sudo vi /etc/prometheus/prometheus.yml

```bash
remote_write:
  - url: "http://localhost:8086/api/v1/prom/write?db=prometheus"

remote_read:
  - url: "http://localhost:8086/api/v1/prom/read?db=prometheus"
```

## Reload Service

```sql
systemctl start influxdb
systemctl enable influxdb

echo 'CREATE DATABASE "prometheus"' `|` influx

systemctl start prometheus
systemctl status prometheus

influx
USE prometheus
select * from /.*/ limit 1
```

## Note that

> influx -host ip

> ./influx -ssl -username <Username>-password <Password>-host <Domain name>-port 8086

## Create account

Như với bất kỳ cơ sở dữ liệu nào, sau khi cài đặt, điều đầu tiên cần làm là tạo tài khoản quản trị viên. Điều này có thể được thực hiện bằng cách sử dụng lệnh sau

> curl -XPOST "http://localhost:8086/query" --data-urlencode "q=CREATE USER admin WITH PASSWORD 'password' WITH ALL PRIVILEGES"

Khi tài khoản được tạo, hãy truy cập trình bao InfluxDB bằng cách sử dụng lệnh:

> influx -username 'admin' -password 'password'

> CREATE USER admin WITH PASSWORD '<password>' WITH ALL PRIVILEGES

For example: CREATE USER admin WITH PASSWORD 'PaSsw0rd' WITH ALL PRIVILEGES

Ví dụ: để truy cập dữ liệu được sử dụng trong hướng dẫn này và xem tất cả các cơ sở dữ liệu hiện có, lệnh thực thi sẽ là:

> curl -G http://localhost:8086/query -u admin:password --data-urlencode "q=SHOW DATABASES"

GRANT administrative privileges to an existing user

> GRANT ALL PRIVILEGES TO <username>

REVOKE administrative privileges from an admin user

> REVOKE ALL PRIVILEGES FROM <username>

SHOW all existing users and their admin status

> SHOW USERS

## Enabling the Firewall

> sudo ufw allow 8086/tcp

## Doc

- https://docs.influxdata.com/influxdb/v1.8/supported_protocols/prometheus/

- https://www.alibabacloud.com/help/en/time-series-database/latest/connect-to-tsdb-for-influxdb-by-using-the-influx-cli



## InfluxDB 2

> wget -qO- https://repos.influxdata.com/influxdb.key `|` gpg --dearmor `|` sudo tee /etc/apt/trusted.gpg.d/influxdb.gpg > /dev/null

> export DISTRIB_ID=$(lsb_release -si); export DISTRIB_CODENAME=$(lsb_release -sc)

> echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdb.gpg] https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" `|` sudo tee /etc/apt/sources.list.d/influxdb.list > /dev/null

> apt-get update

> apt-get install influxdb2

Check

> dpkg -L influxdb2

> influx version

> systemctl start/stop/restart/enable influxdb

Setup influx 

> influx setup

```context
> Welcome to InfluxDB 2.0!
? Please type your primary username cyberithub
? Please type your password *********
? Please type your password again *********
? Please type your primary organization name CyberITHub
? Please type your primary bucket name cyberithub_bucket
? Please type your retention period in hours, or 0 for infinite 0
? Setup with these parameters?
Username: cyberithub
Organization: CyberITHub
Bucket: cyberithub_bucket
Retention Period: infinite
Yes
User Organization Bucket
cyberithub CyberITHub cyberithub_bucket
```
