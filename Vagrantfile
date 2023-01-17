Vagrant.configure("2") do |config|
  config.vm.box = "debian/bullseye64"
  config.vm.synced_folder "giteaConfig", "/home/vagrant/gitea/gitea/custom/conf", type: "nfs", nfs_version: 4
  config.vm.provider "libvirt" do |v|
    v.memory = 4096
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
    mkdir -p /home/vagrant/gitea/{gitea,mysql}
    cd /home/vagrant/gitea
    touch Dockerfile
    echo '# Build stage for Gitea
FROM golang:1.19-alpine3.17 AS build

RUN apk add nodejs npm git make go-bindata g++ && \
    mkdir -p /go/src/gitea && \
    cd /go/src/gitea && \
    git clone https://github.com/go-gitea/gitea && \
    cd gitea && \
    git branch -a && \
    git checkout v1.18.0 && \
    TAGS="bindata" make build

# Deployment stage
FROM alpine:latest AS final

# Copy Gitea binary
COPY --from=build /go/src/gitea/gitea/gitea /usr/local/bin/gitea

RUN apk add git && \
    adduser -D gitea -u 1000

USER gitea

CMD GITEA_WORK_DIR=/data gitea web

EXPOSE 3306 3000 22

    ' > Dockerfile
    docker build --target final -t semlan/gitea:v1 .
    touch docker-compose.yml
    echo 'version: "3"
networks:
  gitea:
    external: false
services:
  server:
    image: semlan/gitea:v1
    container_name: gitea
    environment:
      - USER_UID=1000
      - USER_GID=1000
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
    chown -hR 1000:1000 gitea mysql
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
    docker image prune -a -f
    echo "Machine successfully provisioned at $(date)!"
    echo "Tip: Make sure to add '$(ip a show eth0 | grep 'inet ' | awk '{print $2}' | cut -f1 -d '/')   semlangitea.local' to your /etc/hosts file to navigate to the machine via web browser."
  SHELL
end
