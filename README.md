# forge-server-utils

[![build status](https://travis-ci.org/petrbroz/forge-server-utils.svg?branch=master)](https://travis-ci.org/petrbroz/forge-server-utils)
[![npm version](https://badge.fury.io/js/forge-server-utils.svg)](https://badge.fury.io/js/forge-server-utils)
![node](https://img.shields.io/node/v/forge-server-utils.svg)
![npm downloads](https://img.shields.io/npm/dw/forge-server-utils.svg)
![platforms](https://img.shields.io/badge/platform-windows%20%7C%20osx%20%7C%20linux-lightgray.svg)
[![license](https://img.shields.io/badge/license-MIT-blue.svg)](http://opensource.org/licenses/MIT)

Unofficial tools for accessing [Autodesk Forge](https://developer.autodesk.com/) APIs from Node.js applications
and from browsers, built using [TypeScript](https://www.typescriptlang.org) and modern language features like
[async/await](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function)
or [generators](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function*).

![Autodesk Forge](docs/logo.png)

## Usage

### Server Side

The TypeScript implementation is transpiled into CommonJS JavaScript module with type definition files,
so you can use it both in Node.js projects, and in TypeScript projects:

```js
// JavaScript
const { DataManagementClient } = require('forge-server-utils');
```

```ts
// TypeScript
import {
	DataManagementClient,
	IBucket,
	IObject,
	IResumableUploadRange,
	DataRetentionPolicy
} from 'forge-server-utils';
```

#### Authentication

If you need to generate [2-legged tokens](https://forge.autodesk.com/en/docs/oauth/v2/tutorials/get-2-legged-token)
manually, you can use the `AuthenticationClient` class:

```js
const { AuthenticationClient } = require('forge-server-utils');
const { FORGE_CLIENT_ID, FORGE_CLIENT_SECRET } = process.env;
const auth = new AuthenticationClient(FORGE_CLIENT_ID, FORGE_CLIENT_SECRET);
const authentication = await auth.authenticate(['bucket:read', 'data:read']);
console.log('2-legged token', authentication.access_token);
```

Other API clients in this library are typically configured using a simple JavaScript object
containing either `client_id` and `client_secret` properties (for 2-legged authentication),
or a single `token` property (for authentication using a pre-generated access token):

```js
const { DataManagementClient, BIM360Client } = require('forge-server-utils');
const dm = new DataManagementClient({ client_id: '...', client_secret: '...' });
const bim360 = new BIM360Client({ token: '...' });
```

#### Data Management

```js
const { DataManagementClient } = require('forge-server-utils');
const { FORGE_CLIENT_ID, FORGE_CLIENT_SECRET } = process.env;
const data = new DataManagementClient({ client_id: FORGE_CLIENT_ID, client_secret: FORGE_CLIENT_SECRET });

const buckets = await data.listBuckets();
console.log('Buckets', buckets.map(bucket => bucket.bucketKey).join(','));

const objects = await data.listObjects('foo-bucket');
console.log('Objects', objects.map(object => object.objectId).join(','));
```

#### Model Derivatives

```js
const { ModelDerivativeClient } = require('forge-server-utils');
const { FORGE_CLIENT_ID, FORGE_CLIENT_SECRET } = process.env;
const derivatives = new ModelDerivativeClient({ client_id: FORGE_CLIENT_ID, client_secret: FORGE_CLIENT_SECRET });
const job = await derivatives.submitJob('<your-document-urn>', [{ type: 'svf', views: ['2d', '3d'] }]);
console.log('Job', job);
```

#### Design Automation

```js
const { DesignAutomationClient } = require('forge-server-utils');
const { FORGE_CLIENT_ID, FORGE_CLIENT_SECRET } = process.env;
const client = new DesignAutomationClient({ client_id: FORGE_CLIENT_ID, client_secret: FORGE_CLIENT_SECRET });
const bundles = await client.listAppBundles();
console.log('App bundles', bundles);
```

#### SVF

This project also provides additional utilities for parsing _SVF_
(the proprietary file format used by the Forge Viewer), either from Model Derivative
service, or from local folder.

```js
const { ModelDerivativeClient, ManifestHelper } = require('forge-server-utils');
const { Parser } = require('forge-server-utils/dist/svf');
const { FORGE_CLIENT_ID, FORGE_CLIENT_SECRET } = process.env;
const URN = 'dX...';
const AUTH = { client_id: FORGE_CLIENT_ID, client_secret: FORGE_CLIENT_SECRET };

// Find SVF derivatives in a Model Derivative URN
const modelDerivativeClient = new ModelDerivativeClient(AUTH);
const helper = new ManifestHelper(await modelDerivativeClient.getManifest(URN));
const derivatives = helper.search({ type: 'resource', role: 'graphics' });

// Parse each derivative
for (const derivative of derivatives.filter(d => d.mime === 'application/autodesk-svf')) {
    const parser = await Parser.FromDerivativeService(URN, derivative.guid, AUTH);
    // Enumerate fragments with an async iterator
    for await (const fragment of parser.enumerateFragments()) {
        console.log(JSON.stringify(fragment));
    }
    // Or collect all fragments in memory
    console.log(await parser.listFragments());
}
```

```js
const { Parser } = require('forge-server-utils/dist/svf');

// Parse SVF from local folder
const parser = await Parser.FromFileSystem('foo/bar.svf');
// Parse the property database and query properties of root object
const propdb = await parser.getPropertyDb();
console.log(propdb.findProperties(1));
```

Or just download all SVFs (in parallel) of given model to your local folder:

```js
const { ModelDerivativeClient } = require('forge-server-utils');
const { downloadViewables } = require('forge-server-utils/dist/svf');
const { FORGE_CLIENT_ID, FORGE_CLIENT_SECRET } = process.env;

const modelDerivativeClient = new ModelDerivativeClient({ client_id: FORGE_CLIENT_ID, client_secret: FORGE_CLIENT_SECRET });
await downloadViewables('your model urn', 'path/to/output/folder', modelDerivativeClient);
```

### Client Side (experimental)

The transpiled output from TypeScript is also bundled using [webpack](https://webpack.js.org),
so you can use the same functionality in a browser. There is a caveat, unfortunately: at the moment
it is not possible to request Forge access tokens from the browser
due to [CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) limitations,
so when creating instances of the various clients, instead of providing client ID and secret
you will have to provide the token directly.

```html
<script src="https://cdn.jsdelivr.net/npm/forge-server-utils/dist/browser/forge-server-utils.js"></script>
<script>
	const data = new forge.DataManagementClient({ token: '<your access token>' });
	const deriv = new forge.ModelDerivativeClient({ token: '<your access token>' });
	data.listBuckets()
		.then(buckets => { console.log('Buckets', buckets); })
		.catch(err => { console.error('Could not list buckets', err); });
	deriv.submitJob('<your document urn>', [{ type: 'svf', views: ['2d', '3d'] }])
		.then(job => { console.log('Translation job', job); })
		.catch(err => { console.error('Could not start translation', err); });
</script>
```

Note that you can also request a specific version of the library from CDN by appending `@<version>`
to the npm package name, for example, `https://cdn.jsdelivr.net/npm/forge-server-utils@4.0.0/dist/browser/forge-server-utils.js`.

## Testing

```bash
export FORGE_CLIENT_ID=<your-client-id>
export FORGE_CLIENT_SECRET=<your-client-secret>
export FORGE_BUCKET=<your-test-bucket>
export FORGE_MODEL_URN=<testing-model-urn>
yarn run build # Transpile TypeScript into JavaScript
yarn test
```
