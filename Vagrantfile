Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"
  config.vm.provider "libvirt" do |v|
    v.memory = 1024
    v.cpus = 2
  end
  config.vm.provision "shell", inline: <<-SHELL
    echo "Provisioning machine..."
    apt update
    apt install -y ca-certificates curl gnupg lsb-release
    mkdir -p /etc/apt/keyrings
    curl -fsSL https://download.docker.com/linux/debian/gpg | gpg --dearmor -o /etc/apt/keyrings/docker.gpg
    echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/debian/ $(lsb_release -cs) stable" | tee /etc/apt/sources.list.d/docker.list > /dev/null
    apt update
    apt install -y docker-ce docker-ce-cli containerd.io docker-compose docker-compose-plugin nginx ufw
    mkdir /home/vagrant/gitea
    cd /home/vagrant/gitea
    touch docker-compose.yml
    echo 'version: "3"

networks:
  gitea:
    external: false

services:
  server:
    image: gitea/gitea:1.16.5
    container_name: gitea
    environment:
      - USER_UID=1
      - USER_GID=1
      - GITEA__database__DB_TYPE=mysql
      - GITEA__database__HOST=db:3306
      - GITEA__database__NAME=gitea
      - GITEA__database__USER=gitea
      - GITEA__database__PASSWD=gitea
    restart: always
    networks:
      - gitea
    volumes:
      - ./gitea:/data
      - /home/git/.ssh/:/data/git/.ssh
      - /etc/timezone:/etc/timezone:ro
      - /etc/localtime:/etc/localtime:ro
    ports:
      - "127.0.0.1:3000:3000"
      - "127.0.0.1:2222:22"
    depends_on:
      - db
  db:
    image: mysql:8
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=gitea
      - MYSQL_USER=gitea
      - MYSQL_PASSWORD=gitea
      - MYSQL_DATABASE=gitea
    networks:
      - gitea
    volumes:
      - ./mysql:/var/lib/mysql' > docker-compose.yml
    ufw allow "Nginx Full"
    ufw allow "ssh"
    ufw --force enable
    touch /etc/nginx/sites-available/gitea
    echo 'server {
      # Listen for requests on your domain/IP address.
server_name semlangitea.local;

root /var/www/html;

location / {
    # Proxy all requests to Gitea running on port 3000
    proxy_pass http://localhost:3000;
    
    # Pass on information about the requests to the proxied service using headers
    proxy_set_header HOST $host;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
}
  }' > /etc/nginx/sites-available/gitea
    ln -s /etc/nginx/sites-available/gitea /etc/nginx/sites-enabled/gitea
    nginx -t
    systemctl restart nginx
    docker-compose -p gitea up -d
    echo "Machine successfully provisioned at $(date)!"
    echo "Tip: Make sure to add '$(ip a show eth0 | grep 'inet ' | awk '{print $2}' | cut -f1 -d '/')   semlangitea.local' to your /etc/hosts file to navigate to the machine via web browser."
  SHELL
end
