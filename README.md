# Avtomatska postavitev spletne aplikacije FoodTinder

## Opis aplikacije

FoodTinder je spletna aplikacija, ki uporabniku ti naključno izbere 5 izdelkov iz Mercatorjeve spletne trgovine, ki jih uporabnik lahko povleče v desno, če mu je izdelek
vseč ali pa v levo, če mu izdelek ni všeč.

Aplikacija je sestavljena iz naslednjih servisov:
 1. Redis "podatkovna baza"
 2. API
 3. Web scraper
 4. Fontend
 
 ## Opis problema
 
Postavljanje virtualke, za takšno spletno apliakcijo je na roke zelo zamudno, zato sem se odločil, da bom ta zamudni postopek avtomatiziral na dva načina in sicer:
 1. z Vagrantom
 2. z cloud-init in Multipassom
 
## Rešitev 1 ~ Vagrant
 
### Potrebna orodja

Orodja, ki sem jih uporabil za izvedno prve rešitve:
- orodje **Vagrant**
- hipervizor **VirtualBox**

### Konfiguracija virtualke

Za operacijski sistem sem izbral ```ubuntu/focal64```, dodelil sem ji tudi ```4 virtualne cpu-je``` in ```4 GB RAM-a```
Ker je nameščanje vseh paketov vzame veliko časa, sem bil tudi primoran povečati timeout na 500 sekund. Nazadnje sem 
virtualki dodeli ip ```192.168.27.100``` in naredil port-forwarding na dva porta na virtualki. Iz porta ```8080``` (host) na port ```8082``` (guest) kjer bo tekel API
in iz porta ```9090``` (host) na port ```80``` (guest) kjer bo nginx serviral spletno aplikacijo.

```
  # OPERACIJSKI SISTEM
  config.vm.box = "ubuntu/focal64"

  # HARDWARE
  config.vm.provider "virtualbox" do |vb|
    # 4 VIRTUALNI CPU-JI
    vb.cpus = 4 
    # 4 GB RAMA
    vb.memory = "4096"
  end

  config.vm.boot_timeout = 500
  
  # POVEZAVA DO INTERNETA
  config.vm.network :private_network, ip: "192.168.27.100"
  config.vm.network "forwarded_port", guest: 8082, host: 8080 # API
  config.vm.network "forwarded_port", guest: 80, host: 9090 # WEB APP
```

### Namesčanje paketov

Najprej sem spremenil IP naslov DNS serverja, ker v nasprotnem primeru sem dobil napake pri posodobitvi (```Could not resolve 'archive.ubuntu.com'```).
Nato posodobim virtualko in namestim ```net-tools``` paket zaradi testiranja.

``` 
   # UPDATE
   echo "nameserver 8.8.8.8" | tee /etc/resolv.conf > /dev/null
   apt-get update
   
   # INSTALACIJA RAZNIH DODATNIH PAKETOV
   apt-get install -y net-tools
```

Web scraper sem napisal v programskem jeziku python, zato sem moral najprej namestiti python, pip, in vse 
potrebne dependency-je za izvajanje Web scraperja.

```
   # INSTALCIJA PYTHONA, PIP-a IN OSTALIH DEPENDENCIJEV
   apt-get install -y software-properties-common
   add-apt-repository ppa:deadsnakes/ppa
   apt-get install -y python3
   apt-get install -y python3-pip
   pip install beautifulsoup4
```

Ker sem API, napisal v Springboot-u, ki uporablja Javo in Maven, sem tudi to namestil in ju konfiguriral.

```
   # INSTALACIJA JAVE IN MAVENA
   apt-get install -y default-jdk
   apt-get install -y maven
   wget https://dlcdn.apache.org/maven/maven-3/3.8.6/binaries/apache-maven-3.8.6-bin.tar.gz -P /tmp
   tar xf /tmp/apache-maven-*.tar.gz -C /opt
   ln -s /opt/apache-maven-3.8.6 /opt/maven
   touch /etc/profile.d/maven.sh
   chmod +rwx /etc/profile.d/maven.sh
   echo "export JAVA_HOME=/usr/lib/jvm/default-java" | tee -a /etc/profile.d/maven.sh
   echo "export M2_HOME=/opt/maven" | tee -a /etc/profile.d/maven.sh
   echo "export MAVEN_HOME=/opt/maven" | tee -a /etc/profile.d/maven.sh
   echo "export PATH=${M2_HOME}/bin:${PATH}" | tee -a /etc/profile.d/maven.sh
   source /etc/profile.d/maven.sh
```

Potem sem inštaliral REDIS, ki ga uporabljam kot primarno podatkovno bazo, zato sem ga nastavil v ```appendonly``` mode.

```
   # INSTALACIJA REDISA 
   apt install -y lsb-release
   curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
   echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
   apt-get update
   apt-get install -y redis
   sudo service redis-server start
   redis-cli config set appendonly yes
    redis-cli config set save ""
```

Zatem sem inštaliral še ```node-js``` in ```nginx```, ki ju bom v nadaljevanju uporabil za frontend apliacije.

``` 
   # INSTALACIJA NODE-A IN NGINX-A
   curl -sL https://deb.nodesource.com/setup_16.x | sudo -E bash -
   apt-get install -y nodejs   
   apt-get install -y nginx
```

V ```/usr/local/bin``` sem kloiral vse svoje repote iz GitHuba-a (API, Web scraper in frontend).

```   
   # KLONIRANJE REPOTOV IZ GITHUBA (API IN WEB SCRAPER)
   cd /usr/local/bin
   git clone https://github.com/David-api/foodTinderApi.git
   cd /usr/local/bin/foodTinderApi
   git reset --hard f19ee2d77a813761b057269728f484ffe4fa5158
   cd ..
   git clone https://github.com/David-api/foodTinderWebScraper.git
   cd /usr/local/bin/foodTinderWebScraper
   git reset --hard 99511bfaa8162f4dd01994e3b8e4cab2fa7ef1c1
   cd ..
   git clone https://github.com/David-api/foodTinderFrontend.git
```

Nato sem naredi ```.service``` skripte, ki zaženejo API in Web scraper ko service (proces v ozadju) in ju pognal.

```
   # NAREDIMO SERVICE ZA API (IN POZENEMO)
   cd /usr/local/bin/foodTinderApi
   mvn compile
   mvn clean package
   touch /etc/systemd/system/foodTinderApiService.service
   echo "[Unit]" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "Description=foodTinderApiService" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "After=network.target" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "[Service]" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "ExecStart=java -jar /usr/local/bin/foodTinderApi/target/alergenko-api-0.0.1-SNAPSHOT.jar" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "[Install]" | tee -a /etc/systemd/system/foodTinderApiService.service
   echo "WantedBy=multi-user.target" | tee -a /etc/systemd/system/foodTinderApiService.service

   systemctl daemon-reload
   systemctl enable foodTinderApiService.service
   systemctl start foodTinderApiService.service

   # NAREDIMO SERVICE ZA WEB-SCRAER (IN POZENEMO)
   touch /etc/systemd/system/foodTinderApiWebScraper.service
   echo "[Unit]" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "Description=foodTinderWebScraperService" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "After=network.target" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "[Service]" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "ExecStart=python3 /usr/local/bin/foodTinderWebScraper/Main.py" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "[Install]" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service
   echo "WantedBy=multi-user.target" | tee -a /etc/systemd/system/foodTinderApiWebScraper.service


   systemctl daemon-reload
   systemctl enable foodTinderApiWebScraper.service
   systemctl start foodTinderApiWebScraper.service
```

Nazadnje sem še "zbuildal" react aplikacijo, premaknil vse potrebne datoteke v ```/var/www/html``` 
in konfiguriral nginx, tako da je poslušal na portu 80.

``` 
   # NAREDIMO SERVICE ZA FRONTEND (IN POZENEMO)
   cd /usr/local/bin/foodTinderFrontend
   npm install
   npm run build

   rm index.nginx-debian.html
   rm /etc/nginx/sites-enabled/default
   mv /usr/local/bin/foodTinderFrontend/build/* /var/www/html/

   touch /etc/nginx/conf.d/react.conf
   echo "server{"  | tee -a /etc/nginx/conf.d/react.conf
   echo "  listen 80;" | tee -a /etc/nginx/conf.d/react.conf
   echo "  listen [::]:80;" | tee -a /etc/nginx/conf.d/react.conf
   echo "  root /var/www/html;" | tee -a /etc/nginx/conf.d/react.conf
   echo "  location / {" | tee -a /etc/nginx/conf.d/react.conf
   echo "    try_files \\\$uri /index.html;" | tee -a /etc/nginx/conf.d/react.conf
   echo "  }" | tee -a /etc/nginx/conf.d/react.conf
   echo "}" | tee -a /etc/nginx/conf.d/react.conf

    fuser -k 80/tcp
    service nginx start
    service nginx reload
    service nginx start
```

### Poganjanje skripte

Skripto enostavno poženemo s spodnjim ukazom:

```
vagrant up
```

### Rezultat

Če v spletni brskalnik vpišemo ```http://localhost:8080/product/get``` dobimo primer API odgovora s petimi naključnimi izdelki.

![image](https://user-images.githubusercontent.com/58308216/209155756-5a0e821a-ebdf-48d1-94c9-5200d3a0fb9d.png)

Če v spletni brskalnik vpišemo ```http://localhost:9090/``` se nam odpre spletna aplikacija FoodTinder.

![image](https://user-images.githubusercontent.com/58308216/209155896-715f9bed-6f70-4f91-9a5e-849a1466b820.png)


## Rešitev 2 ~ cloud-init + Multipass

V tem delu bom samo opisal razlike med rešitvijo 1 in rešitvijo 2
 
### Potrebna orodja

Orodja, ki sem jih uporabil za izvedno prve rešitve:
- orodje **cloud-init + Multipass**
- hipervizor **VirtualBox**

### Nameščanje paketov

Vsi koraki so identični, razen posodabljanje virtualke, ker sem moral zbrisati ```chache```, da so se potem vsi paketi normalno namestili. 

```
#cloud-config
runcmd:
  # UPDATE
  - sudo apt-get update
  - sudo killall apt apt-get
  - sudo rm /var/lib/apt/lists/lock
  - sudo rm /var/cache/apt/archives/lock
  - sudo rm /var/lib/dpkg/lock*
  - sudo dpkg --configure -a
```

### Poganjanje skripte

Da poženemo skripto moramo najprej terminal zagnati kot administrator. Nato moramo naložiti ```PsTools``` iz [tega naslova](https://learn.microsoft.com/en-us/sysinternals/downloads/pstools)
in datoteko ekstrahirati v mapo ```Downloads```. Zatem moramo spremeniti driver na virtual box z ukazom ```multipass set local.driver=virtualbox``` 

Da kreiramo virtualko poženemo spodnji ukaz:
```
multipass launch -vvvv -n "msi-homework" -c 4 -d 40G -m 4G  --cloud-init cloud-config.yaml --timeout 1500
```

Nazadnje še naredimo port forwrding s spodnjima ukazoma:
```
& $env:USERPROFILE\Downloads\PSTools\PsExec.exe -s $env:VBOX_MSI_INSTALL_PATH\VBoxManage.exe controlvm "msi-homework" natpf1 "myservice1,tcp,,8080,,8082"
```
```
 & $env:USERPROFILE\Downloads\PSTools\PsExec.exe -s $env:VBOX_MSI_INSTALL_PATH\VBoxManage.exe controlvm "msi-homework" natpf1 "myservice2,tcp,,9090,,80"
```
