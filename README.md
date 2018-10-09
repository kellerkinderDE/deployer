# deployer for shopware

This repository contains a [deployer](https://deployer.org/) configuration for shopware.

## Preparations

### Repository
Download the latest [composer.phar](https://getcomposer.org/download/) and [deployer.phar](https://deployer.org/download) to the ``bin`` directory.
Also download the latest [cachetool.phar](https://gordalina.github.io/cachetool/) if you need to reset an OPCache.

The `web` directory will be used as shopware root directory. Place all custom files for your shopware project here, e.g. plugins and custom themes.

We recommend to use [K10rProject](https://github.com/kellerkinderDE/K10rProject) for database migrations and snippet management so this config depends on it. Please add it to your repository if you want to use it. However if you don't need it you will need to remove it from the plugin list in `deploy.php` and remove the command `k10r:snippets:update` in the task `kellerkinder:shopware:config`.

## Configurations

### deploy-hosts.yaml

This file defines the different hosts you may want to deploy to. Adjust those hosts to your needs.
See [deployer documentation](https://deployer.org/docs/hosts) for further information.

### deploy.php

This file defines the basic configuration via deployers `set`-method. Make sure to adjust the following options to your needs:
* `repository`: URL to the repository you want to deploy.
* `bin/php`: Absolute path to the php executable on your server.
* `bin/composer`: Path to composer. Adjust if you do not want to use the composer.phar.
* `shopware_download_path`: URL to the shopware installations package you want to deploy.
* `shopware_update_version`: Shopware version you want to update to. Will be used to check wether the current installations needs to be updated.
* `shopware_update_path`: URL to shopware update package you want to update to.
* `plugins`: An array of plugins that will be installed and updated during deployment. Those plugins need to be present in the codebase.
* `theme_config`: An array of theme configuration settings that will be set during deployment.
* `fcgi_sockets`: An array of fcgi_sockets that will be used to reset the OPCache.

## kellerkinder tasks (deploy-shopware.php)

### kellerkinder:shopware:install

Wrapper task for the basic installations steps:

#### kellerkinder:shopware:download
Downloads the defined shopware installation package, unpacks it and rsyncs it into the shopware root directory.

#### kellerkinder:vendors
Requires some plugins via composer. In this example config those will be [K10rStaging](https://github.com/kellerkinderDE/K10rStaging) and [K10rDeployment](https://github.com/kellerkinderDE/K10rDeployment)

_Note:_ They will not be installed yet.

### kellerkinder:shopware:update
This task will check wether the currently installed shopware version matches the `shopware_update_version` (see above) and will download the shopware update package and update the shop if needed.

### kellerkinder:shopware:configure
This tasks wraps several configuration steps:

#### kellerkinder:shopware:opcache
This will reset the OPCache using `cachetool.phar` and the `fcgi_sockets` specified in the `deploy.php`

#### kellerkinder:shopware:plugins
This will (re)install `K10rDeployment` which gives us some useful commands for the deployment process. Afterwards it deactivates `SwagUpdate` since we do not want anyone to install an update via backend.
And most importantly it installs and updates all plugins specified in the `plugins` variable in `deploy.php`.

This task should also be used to set plugin configurations that should be the same on all stages like this:
```
run("cd {{shopware_public_path}} && {{bin/php}} bin/console sw:plugin:config:set SwagImportExport useCommaDecimal 0");
```

#### kellerkinder:shopware:config
This step will import and update the snippets defined via `K10rProject`. It will also be used to set configurations that should be the same in all stages. Since this is highly individual there are no configs set in this basic configuration.
Feel free to add configuration settings like this:
```
run("cd {{shopware_public_path}} && {{bin/php}} bin/console k10r:config:set disableShopwareStatistics 1");
```
This would disable the internal shopware statistics. For more information on how to set configurations check [K10rDeployment](https://github.com/kellerkinderDE/K10rDeployment).



#### kellerkinder:shopware:cache'
Clears the cache using `sw:cache:clear` and compiles the themes afterwards. Can also be used to warm up the cache which is not activated by default. See code comments.

### kellerkinder:shopware:production and kellerkinder:shopware:staging
These two commands are used to set configurations that differ between the stages.

The staging task will for example install and activate `K10rStaging` which will show a notice that the user is in a staging environment and catch mails.

This task also sets the shop name and title to `DEV!!!` via `k10r:store:update`.

These tasks are usally used to set plugin configurations differing between stage and production, e.g. PayPal in production or sandbox mode. Or to deactivate a tracking plugin that will only be used production.


## Directories on the server
Within your deploy path deployer will create several directories:

* `.dep`: Information about the releases
* `current`: Symlink that will always be linked to the latest release.
* `releases`: This directory contains the last 10 releases. Configure how many releases are kept via `keep_releases` in `deploy-shopware-php`. 
* `shared/media`: Place your media files here. deployer will symlink them into the shopware root directory across all releases.
* `shared/files`: Same as above but for the document and download files.
* `shared/web`: Place your `config.php` here. Also a good place for the `.htaccess` if it's not identical to the one shippied with shopware.

The shared directories and files are configured in `deploy-shopware.php`.

## GitLab CI

Since we use [GitLab CI](https://about.gitlab.com/features/gitlab-ci-cd/) to manage our code and deploy the projects we also ship the `.gitlab-ci.yml.example` which will work perfectly together with this deployer config.

To get this to work you will need some configuration in GitLab:
* Create the SSH_PRIVATE_KEY variable in the GitLab project settings. This variable needs to contain the *private* SSH key with access to the server.
* Add the deployment key with the matching *public* SSH key.
* Add the *public* SSH key to the `.authorized_keys` file of the server.

## Contribution
Feel free to send pull requests if you have any optimizations. They will be highly appreciated.

## License
MIT licensed, see `LICENSE`