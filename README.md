# Server Jenkins

## Setup

* Add ssh githubt credentials
* Add llaor env.list file (from private secrets repo) as secret file named `llaor_env_list`
* Add llaor db.env.list file (from private secrets repo) as secret file named `llaor_db_env_list`
* Add pachatary env.list file (from private secrets repo) as secret file named `pachatary_env_list`
* Add pachatary db.env.list file (from private secrets repo) as secret file named `pachatary_db_env_list`

## Documentation

To understand the job scripts you must have knowledge about the
[server-setup](https://github.com/jordifierro/server-setup)

Here I paste them:

### Blog

[jordifierro.com](https://jordifierro.com)

#### Deploy

Connect to github repo and hook it.

```bash
sudo mkdir -p .jekyll-cache _site ../bundle
sudo docker run -t --volume="$PWD:/srv/jekyll" --volume="$PWD/../bundle:/usr/local/bundle" --env JEKYLL_ENV=production jekyll/jekyll:3.8 bash -c "jekyll build --trace"
sudo rm -rf /var/www/jordifierro.com/
sudo mkdir -p /var/www/jordifierro.com/html
sudo chown -R $USER:$USER /var/www/jordifierro.com/html
sudo mv _site/* /var/www/jordifierro.com/html/
```

### Taddapp

[taddapp.com](https://taddapp.com)

#### Web

##### Deploy

Connect to github repo and hook it.

```bash
sudo rm -rf /var/www/taddapp.com/
sudo mkdir -p /var/www/taddapp.com/html
sudo chown -R $USER:$USER /var/www/taddapp.com/html
sudo mv * /var/www/taddapp.com/html/
```

### Llaor

[llaor.com](https://llaor.com)

#### Api

##### Backup db

Link it to `llaor/api/db.env.list` file.

```bash
eval "$(cat $llaor_db_env_list)" 
date=$(date +%F)
sudo docker exec llaor-postgres pg_dump -U postgres --verbose $LLAOR_DB > $date.dump
sudo aws s3 cp $date.dump s3://llaor/db/$date.dump --profile llaor
sudo aws s3 cp s3://llaor/db/$date.dump s3://llaor/db/latest.dump --profile llaor
```

##### Deploy

Link it to `llaor/api/env.list` file.

```bash
cat $llaor_env_list > env.list

sudo docker build -t llaor/api .

#sudo docker update --restart=no llaor-nginx-01
#sudo docker stop llaor-nginx-01
#sudo docker rm llaor-nginx-01
#sudo docker update --restart=no llaor-api-01
#sudo docker stop llaor-api-01
#sudo docker rm llaor-api-01
sudo docker run -d --restart=always --env-file env.list --net llaor-net -v llaor-statics-01:/code/llaor/staticfiles --name llaor-api-01 -t llaor/api
sudo docker run --name llaor-nginx-01 -v llaor-statics-01:/usr/share/nginx/html/static:ro -v /etc/nginx/sites-available/api.llaor.com/nginx-01.conf:/etc/nginx/nginx.conf:ro -p 127.0.0.2:80:80 --net llaor-net --restart=always -d nginx

response=000
while [ $response -gt 499 -o "${response}" = 000 ]
do
    sleep 1
    response=$(curl --write-out %{http_code} --silent --output /dev/null 127.0.0.2)
    echo $response
done

sudo docker update --restart=no llaor-nginx-02
sudo docker stop llaor-nginx-02
sudo docker rm llaor-nginx-02
sudo docker update --restart=no llaor-api-02
sudo docker stop llaor-api-02
sudo docker rm llaor-api-02
sudo docker run -d --restart=always --env-file env.list --net llaor-net -v llaor-statics-02:/code/llaor/staticfiles --name llaor-api-02 -t llaor/api
sudo docker run --name llaor-nginx-02 -v llaor-statics-02:/usr/share/nginx/html/static:ro -v /etc/nginx/sites-available/api.llaor.com/nginx-02.conf:/etc/nginx/nginx.conf:ro -p 127.0.0.3:80:80 --net llaor-net --restart=always -d nginx
```


##### Reindex all words

```bash
sudo docker exec -t llaor-api python manage.py reindex_all_words
```

##### Restore db

Link it to `llaor/api/db.env.list` file.

```bash
eval "$(cat $llaor_db_env_list)"
sudo aws s3 cp s3://llaor/db/latest.dump latest.dump --profile llaor
sudo docker run --rm -v $PWD:/src -v llaor-pgdata:/dest -w /src alpine cp latest.dump /dest
sudo docker exec -t llaor-postgres psql -U postgres -c "SELECT pid, pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '$LLAOR_DB' AND pid <> pg_backend_pid();"
sudo docker exec -t llaor-postgres psql -U postgres -c "drop database $LLAOR_DB"
sudo docker exec -t llaor-postgres psql -U postgres -c "create database $LLAOR_DB with owner $LLAOR_DB_ROLE"
sudo docker exec -t llaor-postgres bash -c "psql -U $LLAOR_DB_ROLE -d $LLAOR_DB < /var/lib/postgresql/data/latest.dump"
```

##### Test

```bash
sudo docker-compose down
sudo docker-compose build
sudo docker-compose run api bash -c "pytest && python manage.py test --exclude-tag=elasticsearch"
sudo docker-compose down
```

##### Test and deploy (pipeline)

```bash
stage('test') {
    build 'test';
}
stage('deploy') {
    build 'deploy'
}
```

##### Test and deploy trigger

Hook it to github and make it trigger test_and_deploy (previous one) job.

#### Web

##### Deploy

Connect to github repo and hook it.

```bash
echo 'NODE_PATH=src' > .env
echo 'REACT_APP_API_HOST=https://api.llaor.com' >> .env

cat $llaor_firebase > public/firebase.js

sudo docker build -t llaor/web .
sudo rm -rf build
sudo mkdir build
sudo docker run -v $(pwd)/build/:/usr/src/app/build/ -t llaor/web bash -c "npm run build"
sudo rm -rf /var/www/llaor.com/
sudo mkdir -p /var/www/llaor.com/html
sudo chown -R $USER:$USER /var/www/llaor.com/html
sudo mv build/* /var/www/llaor.com/html/
```


### Pachatary

[pachatary.com](https://pachatary.com)

#### Api

##### Backup db

Link it to `pachatary/api/db.env.list` file.

```bash
eval "$(cat $pachatary_db_env_list)" 
date=$(date +%F)
sudo docker exec pachatary-postgres pg_dump -U postgres --verbose $PACHATARY_DB > $date.dump
sudo aws s3 cp $date.dump s3://pachatary-db/$date.dump --profile pachatary
sudo aws s3 cp s3://pachatary-db/$date.dump s3://pachatary-db/latest.dump --profile pachatary
```

##### Deploy

Link it to `pachatary/api/env.list` file.

```bash
cat $pachatary_env_list > env.list

sudo docker build -t pachatary/api .

sudo docker update --restart=no pachatary-nginx-01
sudo docker stop pachatary-nginx-01
sudo docker rm pachatary-nginx-01
sudo docker update --restart=no pachatary-api-01
sudo docker stop pachatary-api-01
sudo docker rm pachatary-api-01
sudo docker run -d --restart=always --env-file env.list --net pachatary-net -v pachatary-statics-01:/code/pachatary/staticfiles --name pachatary-api-01 -e INTERNAL_IP=127.0.1.1 -t pachatary/api
sudo docker run --name pachatary-nginx-01 -v pachatary-statics-01:/usr/share/nginx/html/static:ro -v /etc/nginx/sites-available/api.pachatary.com/nginx-01.conf:/etc/nginx/nginx.conf:ro -p 127.0.1.1:80:80 --net pachatary-net --restart=always -d nginx

response=000
while [ $response -gt 499 -o "${response}" = 000 ]
do
    sleep 1
    response=$(curl --write-out %{http_code} --silent --output /dev/null 127.0.1.1)
    echo $response
done

sudo docker update --restart=no pachatary-nginx-02
sudo docker stop pachatary-nginx-02
sudo docker rm pachatary-nginx-02
sudo docker update --restart=no pachatary-api-02
sudo docker stop pachatary-api-02
sudo docker rm pachatary-api-02
sudo docker run -d --restart=always --env-file env.list --net pachatary-net -v pachatary-statics-02:/code/pachatary/staticfiles --name pachatary-api-02 -e INTERNAL_IP=127.0.1.2 -t pachatary/api
sudo docker run --name pachatary-nginx-02 -v pachatary-statics-02:/usr/share/nginx/html/static:ro -v /etc/nginx/sites-available/api.pachatary.com/nginx-02.conf:/etc/nginx/nginx.conf:ro -p 127.0.1.2:80:80 --net pachatary-net --restart=always -d nginx
```

##### Reindex all experiences

```bash
sudo docker exec -t pachatary-api python manage.py reindex_all_experiences
```

##### Restore db

Link it to `pachatary/api/db.env.list` file.

```bash
eval "$(cat $pachatary_db_env_list)" 
sudo aws s3 cp s3://pachatary-db/latest.dump latest.dump --profile pachatary
sudo docker run --rm -v $PWD:/src -v pachatary-pgdata:/dest -w /src alpine cp latest.dump /dest
sudo docker exec -t pachatary-postgres psql -U postgres -c "SELECT pid, pg_terminate_backend(pid) FROM pg_stat_activity WHERE datname = '$PACHATARY_DB' AND pid <> pg_backend_pid();"
sudo docker exec -t pachatary-postgres psql -U postgres -c "drop database $PACHATARY_DB"
sudo docker exec -t pachatary-postgres psql -U postgres -c "create database $PACHATARY_DB with owner $PACHATARY_DB_ROLE"
sudo docker exec -t pachatary-postgres bash -c "psql -U $PACHATARY_DB_ROLE -d $PACHATARY_DB < /var/lib/postgresql/data/latest.dump"
```

##### Test

```bash
sudo docker-compose down
sudo docker-compose build
sudo docker-compose run api bash -c "pytest && python manage.py test --exclude-tag=elasticsearch"
sudo docker-compose down
```

##### Test and deploy (pipeline)

```bash
stage('test') {
    build 'test';
}
stage('deploy') {
    build 'deploy'
}
```

##### Test and deploy trigger

Hook it to github and make it trigger test_and_deploy (previous one) job.
