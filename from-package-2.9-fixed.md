# วิธีการติดตั้ง CKAN 2.9 จาก Package บน Ubuntu 18.04 และ 20.04

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
sudo apt-get install openjdk-11-jdk solr-tomcat
```
ให้ทำการแก้ไขโดยคำสั่งดังนี้
```sh
sudo mv /etc/systemd/system/tomcat9.d /etc/systemd/system/tomcat9.service.d
sudo systemctl daemon-reload
```

### 4. ติดตั้ง Package ของ Ubuntu ที่ CKAN ต้องการ:
- สำหรับ Ubuntu 20.04:
```sh
sudo apt-get install -y libpq5 redis-server nginx supervisor libpython2.7 git

sudo add-apt-repository universe

sudo apt install python2

sudo update-alternatives --install /usr/bin/python python /usr/bin/python2 1

sudo update-alternatives --config python

curl https://bootstrap.pypa.io/2.7/get-pip.py --output get-pip.py

sudo python2 get-pip.py
```
- สำหรับ Ubuntu 18.04:
```sh
sudo apt-get install -y libpq5 redis-server nginx supervisor libpython2.7 python-pip git-core
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

### 6. ตั้งค่า Nginx และ Storage path:
```sh
#ตั้งค่า Nginx
wget https://gitlab.nectec.or.th/opend/installing-ckan/-/raw/master/config/nginx/ckan_default.conf -P ./nginx

sudo cp ./nginx/ckan_default.conf /etc/nginx/conf.d/ckan_default.conf

#เตรียม proxycache
sudo mkdir -p /var/cache/nginx/proxycache && sudo chown www-data /var/cache/nginx/proxycache

#เตรียม storage path
sudo mkdir -p /var/lib/ckan/default

sudo chown -R www-data:www-data /var/lib/ckan && sudo chmod -R 775 /var/lib/ckan
```

### 7. ดาวน์โหลดและติดตั้ง CKAN package ตามเวอร์ชั่นของ Ubuntu:
ตรวจสอบเวอร์ชั่นของ Ubuntu โดยใช้คำสั่ง 
```sh
cat /etc/os-release
```
- สำหรับ Ubuntu 20.04:
```sh
    wget http://packaging.ckan.org/python-ckan_2.9-py2-focal_amd64.deb
    sudo dpkg -i python-ckan_2.9-py2-focal_amd64.deb
```
- สำหรับ Ubuntu 18.04:
```sh
    wget http://packaging.ckan.org/python-ckan_2.9-bionic_amd64.deb
    sudo dpkg -i python-ckan_2.9-bionic_amd64.deb
```

### 8. ตั้งค่าและสร้างฐานข้อมูลสำหรับ CKAN
#### 8.1 ตั้งค่า who.ini:
```sh
sudo mv /etc/ckan/default/who.ini /etc/ckan/default/who.ini.bak

sudo ln -s /usr/lib/ckan/default/src/ckan/who.ini /etc/ckan/default/who.ini
```
#### 8.2 แก้ไขไฟล์ config และสร้างฐานข้อมูล CKAN ดังนี้:
```sh
sudo vi /etc/ckan/default/ckan.ini
    - แก้ไข {password1} (จากการตั้งค่าในขั้นตอนที่ 2) ของ sqlalchemy.url
        > sqlalchemy.url = postgresql://ckan_default:{password1}@localhost/ckan_default
    - เปิดการใช้งาน และแก้ไข {password1} (จากการตั้งค่าในขั้นตอนที่ 2) ของ ckan.datastore.write_url
        > ckan.datastore.write_url = postgresql://ckan_default:{password1}@localhost/datastore_default
    - เปิดการใช้งาน และแก้ไข {password2} (จากการตั้งค่าในขั้นตอนที่ 2) ของ ckan.datastore.read_url
        > ckan.datastore.read_url = postgresql://datastore_default:{password2}@localhost/datastore_default
    - กำหนด ip หรือ domain name ที่ ckan.site_url
        > ckan.site_url = http://{ip address}
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

### 9. ปรับแก้ไขสิทธิ์ที่จำเป็น:
```sh
sudo rm -rf /etc/nginx/sites-enabled/ckan

sudo chown -R `whoami` /usr/lib/ckan/default

sudo chmod -R 775 /usr/lib/ckan/default/src/ckan/ckan/public

sudo chown -R www-data:www-data /usr/lib/ckan/default/src/ckan/ckan/public
```

### 10. สร้าง CKAN SysAdmin และกำหนดสิทธิ์ DataStore:
```sh
. /usr/lib/ckan/default/bin/activate

cd /usr/lib/ckan

pip install --upgrade pip

#เปลี่ยน {username}
ckan -c /etc/ckan/default/ckan.ini sysadmin add {username}

#กำหนดสิทธิ์ DataStore
ckan -c /etc/ckan/default/ckan.ini datastore set-permissions | sudo -u postgres psql --set ON_ERROR_STOP=1

deactivate

sudo supervisorctl reload

sudo service nginx restart
```

### 11. ทดสอบเรียกใช้เว็บไซต์ผ่าน http://{ip address} และ login ด้วย SysAdmin


### 12. cronjob สำหรับ page view tracking:
```sh
crontab -e
```
เพิ่มคำสั่งต่อไปนี้

    @hourly /usr/lib/ckan/default/bin/ckan -c /etc/ckan/default/ckan.ini tracking update && /usr/lib/ckan/default/bin/ckan -c /etc/ckan/default/ckan.ini search-index rebuild -r

### 13. การแก้ไขปัญหาการ download file จาก DataStore
เฉพาะกรณีตรวจสอบพบปัญหา หรือที่ตรวจสอบพบว่ามีไฟล์ /usr/lib/ckan/default/src/ckan/ckanext/datastore/blueprint.py
```sh
mv /usr/lib/ckan/default/src/ckan/ckanext/datastore/blueprint.py /usr/lib/ckan/default/src/ckan/ckanext/datastore/blueprint.py.bak

wget https://gitlab.nectec.or.th/opend/installing-ckan/-/raw/master/config/datastore/blueprint.py /usr/lib/ckan/default/src/ckan/ckanext/datastore/

sudo supervisorctl reload
```

### 14. ติดตั้งและตั้งค่า [CKAN Extensions](ckan-extension.md)
