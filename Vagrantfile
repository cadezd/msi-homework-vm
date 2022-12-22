# The most common configuration options are documented and commented below.
# For a complete reference, please see the online documentation at
# https://docs.vagrantup.com.

# Every Vagrant development environment requires a box. You can search for
# boxes at https://vagrantcloud.com/search.

Vagrant.configure("2") do |config|
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
  # Create a forwarded port mapping which allows access to a specific port
  # within the machine from a port on the host machine. In the example below,
  # accessing "localhost:8080" will access port 8082 on the guest machine.
  # NOTE: This will enable public access to the opened port
  config.vm.network :private_network, ip: "192.168.27.100"
  config.vm.network "forwarded_port", guest: 8082, host: 8080 # API
  config.vm.network "forwarded_port", guest: 80, host: 9090 # WEB APP
  

  # Share an additional folder to the guest VM. The first argument is
  # the path on the host to the actual folder. The second argument is
  # the path on the guest to mount the folder. And the optional third
  # argument is a set of non-required options.
  # config.vm.synced_folder "../data", "/vagrant_data"


  # INSTALACIJA VSEGA POTREBNEGA
  config.vm.provision "shell", inline: <<-SHELL
    # UPDATE
    echo "nameserver 8.8.8.8" | tee /etc/resolv.conf > /dev/null
    apt-get update

    # INSTALACIJA RAZNIH DODATNIH PAKETOV
    apt-get install -y net-tools

    # INSTALCIJA PYTHONA, PIP-a IN OSTALIH DEPENDENCIJEV
    apt-get install -y software-properties-common
    add-apt-repository ppa:deadsnakes/ppa
    apt-get install -y python3
    apt-get install -y python3-pip
    pip install beautifulsoup4

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
  

    # INSTALACIJA REDISA 
    apt install -y lsb-release
    curl -fsSL https://packages.redis.io/gpg | sudo gpg --dearmor -o /usr/share/keyrings/redis-archive-keyring.gpg
    echo "deb [signed-by=/usr/share/keyrings/redis-archive-keyring.gpg] https://packages.redis.io/deb $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/redis.list
    apt-get update
    apt-get install -y redis
    sudo service redis-server start
    redis-cli config set appendonly yes
    redis-cli config set save ""


    # INSTALACIJA NODE-A IN NGINX-A
    curl -sL https://deb.nodesource.com/setup_16.x | sudo -E bash -
    apt-get install -y nodejs   
    apt-get install -y nginx

    
    # KLONIRANJE REPOTOV IZ GITHUBA (API IN WEB SCRAPER)
    cd /usr/local/bin
    git clone https://github.com/David-api/foodTinderApi.git
    git clone https://github.com/David-api/foodTinderWebScraper.git
    git clone https://github.com/David-api/foodTinderFrontend.git
    
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

  SHELL
end    