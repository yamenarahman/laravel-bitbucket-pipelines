# laravel-bitbucket-pipelines
Follow these steps to auto test and auto deploy your code with bitbucket pipelines and deployments.

##### 1- Create this file in your repo root. bitbucket.pipelines.yml (repo)
```
# This is a sample build configuration for PHP.
# Check our guides at https://confluence.atlassian.com/x/e8YWN for more examples.
# Only use spaces to indent your .yml configuration.
# -----
# You can specify a custom docker image from Docker Hub as your build environment.
image: php:7.2-fpm

pipelines:
  custom:
    Deploy:
      - step:
          name: Deploy to production
          deployment: production
          script:
            #Install ssh
            - apt-get update -y
            - apt-get install -y ssh
            - cat ./deploy.sh | ssh user@host
            - echo "Deploy step finished"
  branches:
    "dev":
      - step:
          name: Test
          artifacts:
            - storage/**
            - vendor/**
            - public/**
            - .env
          script:
            #Update Image
            - apt-get update

            #Install Zip
            - apt-get install -qy zlib1g-dev zip unzip
            - docker-php-ext-install zip

            #Install Git, Curl and ssh
            - apt-get install -qy git
            - apt-get install -qy curl
            - apt-get install -qy ssh

            #Install MySql
            - apt-get install -qy default-mysql-client
            - docker-php-ext-install pdo_mysql

            #Install Crypt
            - apt-get install -qy libmcrypt-dev
            - yes | pecl install mcrypt-1.0.1

            #Install Composer Platform Reqs (libjpeg, libpng, bcmath, gd and pcntl)
            - apt-get install -qy libjpeg-dev libpng-dev libfreetype6-dev
            - docker-php-ext-install bcmath
            - docker-php-ext-configure gd --with-freetype-dir=/usr/include/ --with-jpeg-dir=/usr/include/
            - docker-php-ext-install -j$(nproc) gd
            - docker-php-ext-install pcntl

            #Copy Environment File
            - ln -f -s .env.pipelines .env

            #Install Composer
            - curl -sS https://getcomposer.org/installer | php -- --install-dir=/usr/local/bin --filename=composer
            - composer install

            #Generate key & Migrate Database
            #- php artisan key:generate
            #- php artisan migrate

            #Run Test Suite
            - ./vendor/bin/phpunit --verbose
          services:
            - docker
            - mysql
          caches:
            - docker
            - composer
      - step:
          name: Deploy to stagging
          deployment: staging
          script:
            #Install ssh
            - apt-get update -y
            - apt-get install -y ssh
            - cat ./deploy.sh | ssh user@host
            - echo "Deploy step finished"

definitions:
  services:
    mysql:
      image: mysql:5.7
      environment:
        MYSQL_DATABASE: "homestead"
        MYSQL_RANDOM_ROOT_PASSWORD: "yes"
        MYSQL_USER: "homestead"
        MYSQL_PASSWORD: "secret"

```

##### 2- Add env pipelines variables also in your repo root. .env.pipelines (repo)
```
APP_ENV=local
APP_KEY=ThisIsThe32CharacterKeySecureKey

DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_DATABASE=homestead
DB_USERNAME=homestead
DB_PASSWORD=secret
```

##### 3- Push to bitbucket so you can setup pipelines in the repo dashboard.

##### 4- Now for auto deployment, add this file to your repo root. deploy.sh (repo)
```
echo "Deploy script started"
sh pull-project.sh
echo "Deploy script finished execution"
```

##### 5- You will need to setup deployment ssh key to your repo. check this https://confluence.atlassian.com/bitbucket/use-ssh-keys-in-bitbucket-pipelines-847452940.html

##### 6- In your deployment server 'shared/vps/dedicated',  add this file to your home directory pull-project.sh (server)
```
cd ~/marketing-srmg
php artisan down
git pull
composer install --no-dev --prefer-dist --optimize-autoloader
php artisan cache:clear
php artisan config:cache
php artisan route:cache
php artisan migrate --force
php artisan queue:restart
php artisan up
echo 'Deploy finished.'
```

### Note: Pipelines run on every merge to `dev` only, if you want them to run on any branch edit `bitbucket.pipelines.yml` to branches: `default` instead of `dev`.

### Note: Deployment is automated to stagging but not to production as it's custom step, to run production deployment go to bitbucket and run the pipeline manually. If you want to automate deployment to production also, you should change branches: `{dev,master}` or `default` and move the whole block from custom step to a regular step at the end of the file.
