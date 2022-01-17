# วิธีการติดตั้ง CKAN 2.9 จาก Source บน Ubuntu 18.04

### 1. Update Package ของ Ubuntu:
```sh
sudo apt-get update
```

### 2. ติดตั้งและตั้งค่า PostgreSQL:
```sh
sudo apt-get install -y postgresql

# สร้าง postgres user สำหรับเขียน ckan_default, datastore_default 
# ใส่ ***{password1}***
sudo -u postgres createuser -S -D -R -P ckan_default

# สร้างฐานข้อมูล ckan_default
sudo -u postgres createdb -O ckan_default ckan_default -E utf-8

# สร้างฐานข้อมูล datastore_default
sudo -u postgres createdb -O ckan_default datastore_default -E utf-8

# สร้าง postgres user สำหรับอ่าน datastore_default 
# ใส่ ***{password2}***
sudo -u postgres createuser -S -D -R -P -l datastore_default

#ตรวจสอบ database list ให้มี database ckan_default และ datastore_default
sudo -u postgres psql -l
```


### 3. ติดตั้งและตั้งค่า Solr:
1. คำสั่งติดตั้ง package สำหรับ solr จะเป็นดังนี้
```sh
sudo apt-get install openjdk-11-jdk solr-tomcat
```
2. Change default Tomcat port from 8080 to 8983 in file /etc/tomcat9/server.xml
```sh
sudo vim /etc/tomcat9/server.xml
```
Change as the following
```sh
From:

<Connector port="8080" protocol="HTTP/1.1"


To:

<Connector port="8983" protocol="HTTP/1.1"
```
3.  แทนที่ไฟล์ default ของ schema.xml ด้วยการ symlink schema file จาก source ของ CKAN
```sh
sudo mv /etc/solr/conf/schema.xml /etc/solr/conf/schema.xml.bak
sudo ln -s /usr/lib/ckan/default/src/ckan/ckan/config/solr/schema.xml /etc/solr/conf/schema.xml
```
4.  รีสตาร์ท tomcat9
```sh 
sudo service tomcat9 restart

```
5.  จากนั้นไปที่ ```http://<ip address>:8983/solr```หากมี Internal Server Error ดังนี้

```sh
java.io.IOException: Cannot create directory: /var/lib/solr/data/index
```
ให้ทำการแก้ไขโดยคำสั่งดังนี้
```sh
sudo mv /etc/systemd/system/tomcat9.d /etc/systemd/system/tomcat9.service.d
sudo systemctl daemon-reload
sudo service tomcat9 restart
```


### 4. ติดตั้ง Package ของ Ubuntu ที่ CKAN ต้องการ:
- สำหรับ Ubuntu 20.04:
```sh
sudo apt-get install python-dev libpq-dev redis-server git

sudo add-apt-repository universe

sudo apt install python2

sudo update-alternatives --install /usr/bin/python python /usr/bin/python2 1

sudo update-alternatives --config python

curl https://bootstrap.pypa.io/pip/2.7/get-pip.py --output get-pip.py

sudo python2 get-pip.py

sudo apt install virtualenv
```
- สำหรับ Ubuntu 18.04:

```sh
sudo apt-get install python-dev libpq-dev redis-server python-pip python-virtualenv git-core
```

### 5. ตั้งค่า python2 และ pip2:
```sh
#ตรวจสอบเวอร์ชั่นของ python และกำหนดให้เป็นเวอร์ชัน 2.7
python -V
# Python 2.7.x

#ตรวจสอบเวอร์ชั่นของ pip และกำหนดให้เป็นการรันจาก ... (python 2.7)
pip -V
# pip x.x.x from /usr/local/lib/python2.7/dist-packages/pip (python 2.7)
```

### 6. ตั้งค่า CKAN path:
```sh
#เตรียม ckan path
sudo mkdir -p /usr/lib/ckan/default

sudo chown -R `whoami` /usr/lib/ckan/default

#เตรียม storage path
sudo mkdir -p /var/lib/ckan/default

sudo chown -R `whoami` /var/lib/ckan && sudo chmod -R 775 /var/lib/ckan
```

### 7. ติดตั้ง CKAN:
```sh
virtualenv --python=python2 /usr/lib/ckan/default

. /usr/lib/ckan/default/bin/activate

cd /usr/lib/ckan/default

pip install --upgrade pip

pip install setuptools==44.1.0

pip install -e 'git+https://github.com/ckan/ckan.git@ckan-2.9.2#egg=ckan[requirements-py2]'

deactivate
```

### 8. ตั้งค่าและสร้างฐานข้อมูลสำหรับ CKAN
#### 8.1 ตั้งค่า who.ini:
```sh
sudo mkdir -p /etc/ckan/default

sudo ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini

sudo chown -R `whoami` /etc/ckan/
```
#### 8.2 แก้ไขไฟล์ config และสร้างฐานข้อมูล CKAN ดังนี้:
```sh
. /usr/lib/ckan/default/bin/activate

ckan generate config /etc/ckan/default/ckan.ini

sudo vi /etc/ckan/default/ckan.ini
    - แก้ไข {password1} (จากการตั้งค่าในขั้นตอนที่ 2) ของ sqlalchemy.url
        > sqlalchemy.url = postgresql://ckan_default:{password1}@localhost/ckan_default
    - เปิดการใช้งาน และแก้ไข {password1} (จากการตั้งค่าในขั้นตอนที่ 2) ของ ckan.datastore.write_url
        > ckan.datastore.write_url = postgresql://ckan_default:{password1}@localhost/datastore_default
    - เปิดการใช้งาน และแก้ไข {password2} (จากการตั้งค่าในขั้นตอนที่ 2) ของ ckan.datastore.read_url
        > ckan.datastore.read_url = postgresql://datastore_default:{password2}@localhost/datastore_default
    - กำหนด ckan.site_url
        > ckan.site_url = http://localhost:5000
    - เปิดการใช้งาน และแก้ไข solr_url
        > solr_url = http://127.0.0.1:8983/solr
    - เปิดการใช้งาน ckan.redis.url
        > ckan.redis.url = redis://localhost:6379/0
    - แก้ไข ckan.plugins (ให้เหมือนตามนี้)
        > ckan.plugins = stats text_view image_view recline_view resource_proxy datastore webpage_view
    - แก้ไข ckan.views.default_views (ให้เหมือนตามนี้)
        > ckan.views.default_views = image_view text_view recline_view webpage_view
    - เปิดการใช้งานและแก้ไข ckan.storage_path
        > ckan.storage_path = /var/lib/ckan/default

sudo service tomcat9 restart

ckan -c /etc/ckan/default/ckan.ini db init

deactivate

```

### 9. สร้าง CKAN SysAdmin และกำหนดสิทธิ์ DataStore:

```sh
. /usr/lib/ckan/default/bin/activate

#เปลี่ยน {username}
ckan -c /etc/ckan/default/ckan.ini sysadmin add {username}

#กำหนดสิทธิ์ DataStore
ckan -c /etc/ckan/default/ckan.ini datastore set-permissions | sudo -u postgres psql --set ON_ERROR_STOP=1

ckan -c /etc/ckan/default/ckan.ini run
```

### 10. ทดสอบเรียกใช้เว็บไซต์ผ่าน http://localhost:5000 และ login ด้วย SysAdmin

### 11. วิธีการ set CKAN Production 

#### 11.1 ติดตั้งและตั้งค่า uwsgi 
```sh
. /usr/lib/ckan/default/bin/activate

pip install uwsgi

deactivate

sudo cp /usr/lib/ckan/default/src/ckan/ckan-uwsgi.ini /etc/ckan/default/

sudo cp /usr/lib/ckan/default/src/ckan/wsgi.py /etc/ckan/default/
```
#### 11.2 ติดตั้งและตั้งค่า supervisor สำหรับรัน uwsgi
```sh
sudo apt-get install supervisor

# สร้าง log โฟร์เดอร์สำหรับ CKAN
sudo mkdir -p /var/log/ckan

# สร้าง supervisor config สำหรับ ckan-uwsgi
sudo vi /etc/supervisor/conf.d/ckan-uwsgi.conf
```
เพิ่มคำสั่งต่อไปนี้
```sh
[program:ckan-uwsgi]

command=/usr/lib/ckan/default/bin/uwsgi -i /etc/ckan/default/ckan-uwsgi.ini

; Start just a single worker. Increase this number if you have many or
; particularly long running background jobs.
numprocs=1
process_name=%(program_name)s-%(process_num)02d

; Log files - change this to point to the existing CKAN log files
stdout_logfile=/var/log/ckan/ckan-uwsgi.stdout.log
stderr_logfile=/var/log/ckan/ckan-uwsgi.stderr.log

; Make sure that the worker is started on system start and automatically
; restarted if it crashes unexpectedly.
autostart=true
autorestart=true

; Number of seconds the process has to run before it is considered to have
; started successfully.
startsecs=10

; Need to wait for currently executing tasks to finish at shutdown.
; Increase this if you have very long running tasks.
stopwaitsecs = 600

; Required for uWSGI as it does not obey SIGTERM.
stopsignal=QUIT
```
#### 11.3 ติดตั้งและตั้งค่า nginx
```sh
sudo apt-get install nginx

sudo vi /etc/nginx/sites-available/ckan
```
เพิ่มคำสั่งต่อไปนี้
```sh
proxy_cache_path /var/cache/nginx/proxycache levels=1:2 keys_zone=cache:30m max_size=250m;
proxy_temp_path /tmp/nginx_proxy 1 2;

server {
    client_max_body_size 100M;
    location / {
        proxy_pass http://127.0.0.1:8080/;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $host;
        proxy_cache cache;
        proxy_cache_bypass $cookie_auth_tkt;
        proxy_no_cache $cookie_auth_tkt;
        proxy_cache_valid 30m;
        proxy_cache_key $host$scheme$proxy_host$request_uri;
        # In emergency comment out line to force caching
        # proxy_ignore_headers X-Accel-Expires Expires Cache-Control;
    }

}
```
#### 11.4 เริ่มการใช้งาน CKAN
```sh
# ลบไฟล์ default ของ nginx ออก
sudo rm -r /etc/nginx/sites-enabled/default

# เปิดใช้งาน CKAN สำหรับ ngixn
sudo ln -s /etc/nginx/sites-available/ckan /etc/nginx/sites-enabled/ckan

#เตรียม proxycache
sudo mkdir -p /var/cache/nginx/proxycache && sudo chown www-data /var/cache/nginx/proxycache

#ปรับแก้ไขสิทธิ์ที่จำเป็น
sudo chown -R www-data:www-data /var/lib/ckan

sudo chown -R www-data:www-data /usr/lib/ckan/default/src/ckan/ckan/public

sudo chown -R www-data /tmp/default/

#แก้ไข CKAN config
sudo vi /etc/ckan/default/ckan.ini
    - กำหนด ip ที่ ckan.site_url
        > ckan.site_url = http://{ip address}


# Restart Service
sudo supervisorctl reload

sudo service nginx restart
```

### 12. ทดสอบเรียกใช้เว็บไซต์ผ่าน http://{ip address}

### 13. cronjob สำหรับ page view tracking:

สร้าง background jobs config
```sh
sudo cp /usr/lib/ckan/default/src/ckan/ckan/config/supervisor-ckan-worker.conf /etc/supervisor/conf.d/ckan-worker.conf
```

ทำการ reload supervisor
```sh
sudo supervisorctl reload
```

```sh
crontab -e
```
เพิ่มคำสั่งต่อไปนี้

    @hourly /usr/lib/ckan/default/bin/ckan -c /etc/ckan/default/ckan.ini tracking update && /usr/lib/ckan/default/bin/ckan -c /etc/ckan/default/ckan.ini search-index rebuild -r


### 14. การแก้ไขปัญหาการ download file จาก DataStore
เฉพาะกรณีตรวจสอบพบปัญหา หรือที่ตรวจสอบพบว่ามีไฟล์ /usr/lib/ckan/default/src/ckan/ckanext/datastore/blueprint.py
```sh
mv /usr/lib/ckan/default/src/ckan/ckanext/datastore/blueprint.py /usr/lib/ckan/default/src/ckan/ckanext/datastore/blueprint.py.bak

wget https://gitlab.nectec.or.th/opend/installing-ckan/-/raw/master/config/datastore/blueprint.py /usr/lib/ckan/default/src/ckan/ckanext/datastore/

sudo supervisorctl reload
```
### 15. ติดตั้งและตั้งค่า [CKAN Extensions](ckan-extension.md)



