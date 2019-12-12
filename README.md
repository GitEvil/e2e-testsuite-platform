# E2E Platform Testsuite for Shopware 6

![GitHub](https://img.shields.io/github/license/shopware/e2e-testsuite-platform)
![GitHub last commit](https://img.shields.io/github/last-commit/shopware/e2e-testsuite-platform)
![David](https://img.shields.io/david/shopware/e2e-testsuite-platform)
![GitHub release (latest SemVer)](https://img.shields.io/github/v/release/shopware/e2e-testsuite-platform)

* [Setup for plugins](#setup-for-plugins)
* [Writing tests](#writing-tests)
* [Locally running tests](#locally-running-tests)
* [Gitlab integration](#gitlab-integration)
* [Commands](#commands)
  + [General commands](#general-commands)
  + [Storefront commands](#storefront-commands)
  + [System commands](#system-commands)
  + [API commands](#api-commands)
* [Local development of the testsuite](#local-development-of-the-testsuite)

This package contains the e2e test suite for Shopware 6. The test suite is based on the testing framework [Cypress](https://github.com/cypress-io/cypress) just as these corresponding plugins:

- [`cypress-select-tests`](https://github.com/bahmutov/cypress-select-tests)
- [`cypress-log-to-output`](https://github.com/flotwig/cypress-log-to-output)
- [`cypress-file-upload`](https://github.com/abramenal/cypress-file-upload)

## Setup for plugins

Depending on your environment (administration or storefront) please create a folder structure:

```text
Resources
  `-- app
    `-- <environment>
      `-- test
        `-- e2e
          `-- cypress
            |-- fixtures
            |-- integration
            |-- plugins
            `-- support
```

Navigate to `Resources/app/<environment>/test/e2e` and run `npm init -y` to create a `package.json`. A convenient way to run tests locally is to place a script inside the newly created `package.json`. Simply add the following code section:

```json
"scripts": {
   "open": "node_modules/.bin/cypress open"
},
```

Now install this package using the following command:

```bash
npm install --save github:shopware/e2e-testsuite-platform#master
```

Next up, please create a new file `e2e/cypress/plugins/index.js` with the following content:

```js
module.exports = require('@shopware/e2e-testsuite-platform/cypress/plugins');
```

Lastly, create a new file `e2e/cypress/support/index.js` with the following content:

```js
// Require test suite commons
require('@shopware/e2e-testsuite-platform/cypress/support');
```

## Writing tests

Before you start, head over to the [Cypress documentation](https://docs.cypress.io/guides/getting-started/writing-your-first-test.html#Add-a-test-file) and get familiar with the testing framework. It might also be useful to have a look atour [guide on how to write test using Cypress](https://docs.shopware.com/en/shopware-platform-dev-en/how-to/end-to-end-tests-in-plugins).

## Running tests locally

Switch to the folder `Resources/app/<enviroment>/test/e2e` and execute the following command:

```bash
CYPRESS_baseUrl=<your-url> npm run open
```

This will start the Cypress test runner that lets you run and debug any available test.

## Gitlab integration

In the following configuration, a new job called `.E2E` is created as a template. It first installs Shopware and the plugin, initializes the administration and storefront, sets up a testing database and executes all associated tests.

```yml
.E2E:
    stage: E2E
    dependencies: []
    services:
        -   name: docker:18.09.7-dind
            alias: docker
        -   name: mariadb:10.3
            alias: mysql
    artifacts:
        when: always
        paths:
            - development/build/artifacts/e2e/
        reports:
            junit: development/build/artifacts/e2e/*.xml
    script:
        - ./psh.phar init --APP_ENV="prod"
        - php bin/console plugin:refresh
        - php bin/console plugin:install --activate $PLUGINAME -c
        - ./psh.phar storefront:init --APP_ENV="prod" --DB_NAME="shopware_e2e"
        - ./psh.phar administration:init --APP_ENV="prod"
        - ./psh.phar e2e:dump-db --APP_ENV="prod"
        - chown -R 1000:1000 .
        - docker run --name cypress -d -t --add-host="docker.vm:$(hostname -I)" -e CYPRESS_baseUrl=http://docker.vm:8000 -v $(pwd)/custom/plugins/$PLUGINAME/src/Resources/app/$MODULE/test/e2e:/e2e -v $(pwd):/app cypress/browsers:node10.11.0-chrome75
        - docker exec cypress npm clean-install --prefix /e2e
        - forever start custom/plugins/$PLUGINAME/src/Resources/app/$MODULE/test/e2e/node_modules/@shopware/e2e-testsuite/routes/cypress.js
        - docker exec cypress npx cypress run --project /e2e --browser chrome --config baseUrl=http://docker.vm:8000 --config numTestsKeptInMemory=0 --spec e2e/cypress/integration/**/*
        - docker rm -f cypress

Administration E2E:
    extends: .E2E
    variables:
        MODULE: "administration"
        PLUGINAME: "SwagCustomizedProduct"

Storefront E2E:
    extends: .E2E
    variables:
        MODULE: "storefront"
        PLUGINAME: "SwagCustomizedProduct"
```

At the bottom of the configuration file we created another job called `Administration E2E`. It extends the previously created job `.E2E` and sets up enviroment variables to modify the plugin name as well as the enviroment (administration or storefront).

## Commands

The package contains several pre-built commands for easier navigation in administration and storefront using Cypress.

### General commands

#### Switches administration UI locale to EN_GB

```js
cy.setLocaleToEnGb()
```

#### Manual logging in the Administration

```js
cy.login(userType)
```

#### Types in an input element and checks if the text is correctly displayed.

```js
cy.get('input[name="companyName"]').typeAndCheck('shopware AG');
```

#### Clears a field, then types in an input element and checks if the content is correctly displayed

```js
cy.get('input[name="companyName"]').clearTypeAndCheck('shopware AG');
```

#### Types in a sw-select field and checks if the content is correctly displayed (multi select)

```js
cy.get('.select-payment-method')
  .typeMultiSelectAndCheck('Invoice', {
    searchTerm: 'Invoice'
  });
```

#### Types in an sw-select field (single select).

```js
cy.get('.sw-sales-channel-switch')
  .typeSingleSelect('Storefront', '.sw-entity-single-select');
```

#### Types in an sw-select field and checks if the content is correctly displayed (single select).

```js
cy.get('.sw-sales-channel-switch')
  .typeSingleSelectAndCheck('Storefront', '.sw-entity-single-select');
```

#### Types in an legacy swSelect field and checks if the content is correctly displayed.

```js
cy.get('.sw-settings-shipping-detail__delivery-time')
  .typeLegacySelectAndCheck(
    '1-3 days', {
        searchTerm: '1-3 days'
    }
);
```

#### Types into the global search field and verifies the search term by comparing the urls

```js
cy.get('.sw-search-bar__input').typeAndCheckSearchField('Ruler');
```

#### Waits for a notification to appear and checks its message

```js
cy.awaitAndCheckNotification('Shipping method "Luftpost" has been deleted.');
```

#### Clicks context menu in order to trigger a certain action

```js
cy.clickContextMenuItem(
    '.sw-customer-list__view-action',
    '.sw-context-button__button',
    `.sw-data-grid__row--0`
);
```

#### Navigates to the module by clicking the corresponding main menu item

```js
cy.clickMainMenuItem({
    targetPath: '#/sw/product/index',
    mainMenuId: 'sw-catalogue',
    subMenuId: 'sw-product'
});
```

#### Clicks user menu in order to open it

```js
cy.openUserActionMenu();
```

#### Performs a drag and drop operation

```js
cy.get('.sw-cms-sidebar__block-preview')
  .first()
  .dragTo('.sw-cms-section__empty-stage');
```

### Storefront commands

#### Gets a Sales Channel ID via the Admin API

```js
cy.getSalesChannelId()
```

#### Performs storefront API requests

```js
cy.storefrontApiRequest(method, endpoint, header = {}, body = {})
```

#### Returns a random product including its ID, name and URL

```js
cy.getRandomProductInformationForCheckout()
```

### System commands

#### Activates the Shopware theme for use in the Cypress test runner

```js
cy.activateShopwareTheme();
```

#### Cleans up any previous states by restoring the database and clearing caches

```js
cy.cleanUpPreviousState();
```

#### Initially opens up the administration, waits for a successful "me" call

```js
cy.openInitialPage();
```

### API commands

#### Authenticates towards the Shopware API

```js
cy.authenticate()
```

#### Silently keeps a log using the Shopware API

```js
cy.loginViaApi()
```

#### Searches for an existing entity by using Shopware API at a given endpoint

```js
cy.searchViaAdminApi(data)
```

#### API request handling

```js
cy.requestAdminApi(method, url, requestData)
```

#### Updates an existing entity using Shopware API at a given endpoint

```js
cy.updateViaAdminApi(endpoint, id, data)
```

## Local testsuite development

It's also possible to employ a local clone of the test suite instead of the package here on Github. It opens up the ability to write new commands and / or modify test suite behavior without pushing it to the `master` branch. [`npm link`](https://docs.npmjs.com/cli/link.html) provides a convenient way of doing just that.

```bash
git clone git@github.com:shopware/e2e-testsuite-platform.git
cd e2e-testsuite-platform
npm link

# Switch to the e2e folder inside your project for example:
# custom/plugins/customized-product/src/Resources/app/storefront/test/e2e

npm uninstall # removes the remote copy of the package from github
npm link @shopware/e2e-testsuite-platform
```
