# Mail-API

Rest API for mail content scanning based on NodeJS, Amavisd and Spamassassin

To be used --> v1-redis

<!-- ## Test v2/v3 in docker container

v1->1234 v2->1235 v3->1236

```sh
docker run -it -p 1236:1236 --name <container_name> ubuntu:14.04
```

once in docker container

```sh
rm -vf /var/lib/apt/lists/* && apt-get update && mkdir /data \
apt-get install -y nodejs npm spamassassin curl apt-transport-https ca-certificates \
curl --fail -ssL -o setup-nodejs https://deb.nodesource.com/setup_8.x && bash setup-nodejs && apt-get install -y nodejs build-essential
```

_this will take some time..._

in another terminal

```sh
docker cp path/to/folder <container_name>:/data
```

back inside docker container

```sh
cd /data && npm install \
npm run start-dev
```

the code changed by `docker cp <source> <dest>` will be recognized and server will restart automatically -->

## Run server with docker-compose

- v1 (2 services w/ shared volume and `inotify-tools`)

    ```sh
    cd v1/docker
    ```

- **v1-redis** (same as v1 but w/ redis message queue service for training job)

    ```sh
    cd v1-redis/docker
    ```

- v2 (1 service using `node-cmd` module)

    ```sh
    cd v2
    ```

- v3 (same as v2 but w/ text/plain request body)

    ```sh
    cd v3
    ```

then

```sh
docker-compose down --rm all --volumes --remove-orphans &&
docker-compose build --no-cache &&
docker-compose up -d --force-recreate --remove-orphans
```

to install **with debug** console remove `-d` option from `docker-compose up` command

### (optional) clear unused docker images and containers

```sh
docker rm $(docker ps -aqf status=exited) &&
docker rmi $(docker images -qf dangling=true)
```

_may take time to install..._

## Usage

The API consists of 3 function calls

_**port: v1-1234 v1-redis-1234 v2-1235 v3-1236**_

### Header

| Key | Value |
| --- | --- |
| Content-Type | multipart/form-data |

### Body

- form-data

| Request | Key | Value |
| --- | --- | --- |
| `/spam` | spam | _spam_mail_file_ |
| `/ham` | ham | _ham_mail_file_ |
| `/test` | test | _mail_file_ |

***body as plain text for v3***

### API pattern

- `localhost:1234/spam`
  - mark content as _SPAM_
  - method: POST
- `localhost:1234/ham`
  - mark content as _HAM_
  - method: POST
- `localhost:1234/test`
  - send content to _TEST_ spam score
  - method: PUT

### Extra

***all post method***

- train spamassassin with spam/ham folder at `./mailtest`
  - `localhost:1236/spams` calls `sa-learn --spam 'folder_name'`
  - `localhost:1236/hams` calls `sa-learn --ham 'folder_name'`

    where body: ***`text/plain`***
    ```sh
    {
      "path": <path>
    }
    ```
    As of now, **\<path\>** can be: `easy_ham_1`, `easy_ham_2`, `easy_ham_3`, `hard_ham_1`, `hard_ham_2`, `spam_1`, `spam_2`, `spam_3`
- clear and peek spamassassin trained bayes database
  - `localhost:1236/clear` calls `sa-learn --clear`
  - `localhost:1236/peek` calls `sa-learn --backup`

### Example

```sh
cURL Testing
curl -X PUT -H "Content-Type: multipart/form-data" -H "Authorization: abcd" -F "test=@./mail1.txt" localhost:1234/test
curl -X POST -H "Content-Type: application/json" -H "Authorization: abcd" -d '{"path": "spam_1"}' localhost:1234/spams
curl -X POST -H "Authorization: abcd" localhost:1234/clear

curl -X POST -H "Authorization: abcd" localhost:1234/peek

curl -X POST -F "spam=@./mailtest/spam_1/00378.143069173c8ee0047916f124032367d1" -H "Authorization: abcd" localhost:1234/spam
curl -X POST -F "spam=@./mailtest/spam_1/00252.7e355e0c5fd1de609684544262435579" -H "Authorization: abcd" localhost:1234/spam
curl -X POST -F "spam=@./mailtest/spam_1/00126.e98e1ba87a38e0cceeb55f3b86dbd4dd" -H "Authorization: abcd" localhost:1234/spam
curl -X POST -F "spam=@./mailtest/spam_1/00497.ebf699da617b11135f3aa9173b9781b9" -H "Authorization: abcd" localhost:1234/spam
curl -X POST -F "spam=@./mailtest/spam_1/00239.2f1370f9cba5ab21297eadb2af40b051" -H "Authorization: abcd" localhost:1234/spam

curl -X POST -F "ham=@./mailtest/easy_ham_1/1909.97cbba56dde3e037c13ff2911be560fd" -H "Authorization: abcd" localhost:1234/ham
curl -X POST -F "ham=@./mailtest/easy_ham_1/2538.e3f596927fd5dd74ad23a0792a449241" -H "Authorization: abcd" localhost:1234/ham
curl -X POST -F "ham=@./mailtest/easy_ham_1/0625.42179d43c513cf5c4e9d9ef00e9235a1" -H "Authorization: abcd" localhost:1234/ham
curl -X POST -F "ham=@./mailtest/easy_ham_1/1894.feeadafed8d22fbefccef31f2a2eb74a" -H "Authorization: abcd" localhost:1234/ham
curl -X POST -F "ham=@./mailtest/easy_ham_1/1234.30168dbefeafcbbd7649c9153a182482" -H "Authorization: abcd" localhost:1234/ham

curl -X PUT -H "Content-Type: multipart/form-data" -H "Authorization: abcd" -F "test=@./mailtest/spam_1/00378.143069173c8ee0047916f124032367d1" localhost:1234/test
curl -X PUT -H "Content-Type: multipart/form-data" -H "Authorization: abcd" -F "test=@./mailtest/easy_ham_1/1234.30168dbefeafcbbd7649c9153a182482" localhost:1234/test

** with -m (numberic) for response time **
```
