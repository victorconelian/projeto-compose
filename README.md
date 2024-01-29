## Aplicando Docker Compose

Para fins de estudos Docker, pegamos um repositório de uma aplicação React com backend em NodeJS e banco de dados Mariadb. Nosso principal objetivo será escrever os Dockerfiles de cada serviço e subir essa aplicação usando o Compose, de forma que possamos visualizá-la na porta 80 local.

### Dockerfile:
**backend:**
```
FROM node:lts AS development

# variaveis de ambiente do node
ARG NODE_ENV=production
ENV NODE_ENV $NODE_ENV

WORKDIR /code

# porta padrao 80 do node, 9929 e 9230 (testes) para debug
ARG PORT=80
ENV PORT $PORT
# expondo as portas
EXPOSE $PORT 9229 9230

COPY package.json /code/package.json
COPY package-lock.json /code/package-lock.json
RUN npm ci

# healthcheck a cada 30s
HEALTHCHECK --interval=30s \
  CMD node healthcheck.js

# copy in our source code last, as it changes the most
COPY . /code

CMD [ "node", "src/index.js" ]

FROM development as dev-envs
RUN <<EOF
apt-get update
apt-get install -y --no-install-recommends git
EOF

RUN <<EOF
useradd -s /bin/bash -m vscode
groupadd docker
usermod -aG docker vscode
EOF
COPY --from=gloursdocker/docker / /
```

**frontend:**
```
FROM node:lts AS development

ENV CI=true
ENV PORT=3000

WORKDIR /code
COPY package.json /code/package.json
COPY package-lock.json /code/package-lock.json
RUN npm ci
COPY . /code

CMD [ "npm", "start" ]

FROM development AS builder

RUN npm run build

FROM development as dev-envs
RUN <<EOF
apt-get update
apt-get install -y --no-install-recommends git
EOF

RUN <<EOF
useradd -s /bin/bash -m vscode
groupadd docker
usermod -aG docker vscode
EOF
COPY --from=gloursdocker/docker / /
CMD [ "npm", "start" ]

FROM nginx:1.13-alpine

COPY --from=builder /code/build /usr/share/nginx/html
```

### Estrutura do Projeto:
```
git branch compose
.
├── backend
│   ├── Dockerfile
│   ...
├── db
│   └── password.txt
├── compose.yaml
├── frontend
│   ├── ...
│   └── Dockerfile
└── README.md
```
### Compose:
```
services:
  backend:
    build: backend
    ports:
      - 88:80
      - 9229:9229
      - 9230:9230
    ...
  db:
    # We use a mariadb image which supports both amd64 & arm64 architecture
    image: mariadb:10.6.4-focal
    ...
  frontend:
    build: frontend
    ports:
    - 80:3000
    ...
```
O arquivo Compose define uma aplicação com três serviços, banco de dados, backend e frontend. Quando fizer o deploy da aplicação, o docker compose mapeia a porta 3000 do serviço frontend para a porta 80 do host, como especificado no arquivo. Garante que a porta 80 não esteja em uso.

## Deploy com docker compose

```
$ docker compose up -d
Creating network "react-express-mysql_default" with the default driver
Building backend
Step 1/16 : FROM node:10
 ---> aa6432763c11
...
Successfully tagged react-express-mysql_frontend:latest
WARNING: Image for service frontend was built because it did not already exist. To rebuild this image you must use `docker-compose build` or `docker-compose up --build`.
Creating react-express-mysql_db_1 ... done
Creating react-express-mysql_backend_1 ... done
Creating react-express-mysql_frontend_1 ... done
```

## Resultados Esperados

Lista com os containers que devem aparecer executando e suas portas:
```
$ docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS                   PORTS                                                  NAMES
f3e1183e709e        react-express-mysql_frontend   "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes             0.0.0.0:3000->3000/tcp                                 react-express-mysql_frontend_1
9422da53da76        react-express-mysql_backend    "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes (healthy)   0.0.0.0:80->80/tcp, 0.0.0.0:9229-9230->9229-9230/tcp   react-express-mysql_backend_1
a434bce6d2be        mysql:8.0.19                   "docker-entrypoint.s…"   8 minutes ago       Up 8 minutes             3306/tcp, 33060/tcp                                    react-express-mysql_db_1
```
Após iniciado o deploy, navegue para o `http://localhost:80` no seu navegador web.

![page](./output.png)


Parar e remover os containers:
```
$ docker compose down
Stopping react-express-mysql_frontend_1 ... done
Stopping react-express-mysql_backend_1  ... done
Stopping react-express-mysql_db_1       ... done
Removing react-express-mysql_frontend_1 ... done
Removing react-express-mysql_backend_1  ... done
Removing react-express-mysql_db_1       ... done
Removing network react-express-mysql_default

```