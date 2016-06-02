Mailvelope Keyserver [![Build Status](https://travis-ci.org/mailvelope/keyserver.svg?branch=master)](https://travis-ci.org/mailvelope/keyserver)
==============

A simple OpenPGP public key server that validates email address ownership of uploaded keys.



## Why not use Web of Trust?

There are already OpenPGP key servers like the [SKS keyserver](https://bitbucket.org/skskeyserver/sks-keyserver/wiki/Home) that employ the [Web of Trust](https://en.wikipedia.org/wiki/Web_of_trust) to provide a way to authenticate a user's PGP keys. The problem with these servers are discussed [here](https://en.wikipedia.org/wiki/Key_server_(cryptographic)#Problems_with_keyservers).

### Privacy

The web of trust raises some valid privacy concerns. Not only is a user's social network made public, common SKS servers are also not compliant with the [EU Data Protection Directive](https://en.wikipedia.org/wiki/Data_Protection_Directive) due to lack of key deletion. This key server addresses these issues by not employing the web of trust and by allowing key removal.

### Usability

The main issue with the Web of Trust though is that it does not scale in terms of usability. The goal of this key server is to enable a better user experience for OpenPGP user agents by providing a more reliable source of public keys, where users verify their email address after key upload. This prevents user A from uploading a public key for user B. With this property in place, automatic key lookup is more reliable than with standard SKS servers.

This requires more trust to be placed in the service provider that hosts a key server, but we believe that this trade-off is necessary to improve the user experience for average users. Tech-savvy users or users with a threat model that requires stronger security may still choose to verify PGP key fingerprints just as before.

## Standardization and (De)centralization

The idea is that an identity provider such as an email provider can host their own key server under a common `openpgpkeys` subdomain. An OpenPGP supporting user agent should attempt to lookup keys under the user's domain e.g. `https://openpgpkeys.example.com` for `user@example.com` first. User agents can host their own fallback key server as well, in case a mail provider does not provide its own key directory.



# Api

The key server provides a modern RESTful api over HTTPS, but is also backwards compatible to the [OpenPGP HTTP Keyserver Protocol (HKP)](https://tools.ietf.org/html/draft-shaw-openpgp-hkp-00) for key upload and lookup.

## HKP api

### Upload a new key

```
POST /pks/add
```

#### HTTP Request Headers

```
'Content-Type': 'application/x-www-form-urlencoded; charset=UTF-8'
```

#### Payload (Form data):

* **keytext**: The ascii armored public PGP key to be uploaded

Example:

```
keytext=-----BEGIN PGP PUBLIC KEY BLOCK----- ... -----END PGP PUBLIC KEY BLOCK-----
```

### Lookup a key

Currently only the `get` operation is implemented. Other operations will return a `501` (not implemented) http error code.

#### By key id

```
GET /pks/lookup?op=get&search=0x0123456789ABCDEF
```

#### By email address

```
GET /pks/lookup?op=get&search=user@example.com
```

#### Machine readable

```
GET /pks/lookup?op=get&options=mr&search=user@example.com
```



## REST api

### Upload new key

```
POST /api/v1/key
```

#### Payload (JSON):

* **publicKeyArmored**: The ascii armored public PGP key to be uploaded
* **primaryEmail (optional)**: The ascii armored block is parsed to check for user ids, so this parameter is purely optional. Normally a verification email is sent to every user id found in the pgp key. To prevent this behaviour, user agents can specify the user's primary email address to send out only one email.

Example:

```json
{
  "publicKeyArmored": "-----BEGIN PGP PUBLIC KEY BLOCK----- ... -----END PGP PUBLIC KEY BLOCK-----",
  "primaryEmail": "user@example.com"
}
```

### Verify uploaded key

```
GET /api/v1/verify?keyid=0123456789ABCDEF&nonce=123e4567-e89b-12d3-a456-426655440000
```

### Lookup a key

#### By key id

```
GET /api/v1/key?keyid=0123456789ABCDEF
```

#### By email address

```
GET /api/v1/key?email=user@example.com
```

#### By email address (shorthand link for sharing)

```
GET /user/user@example.com
```

### Request key removal

#### By key id

```
DELETE /api/v1/key?keyid=0123456789ABCDEF
```

#### By email address

```
DELETE /api/v1/key?email=user@example.com
```

### Verify key removal

```
GET /api/v1/verifyRemove?keyid=0123456789ABCDEF&nonce=123e4567-e89b-12d3-a456-426655440000
```


# Development

Copy the `credentials.json` file into the git repo root directory. This file can be changed to configure a local development installation:

```shell
cp ./res/credentials.json .
```

## Requirements

The server is written is in JavaScript ES6 and runs on [Node.js](https://nodejs.org/) v4+. It uses [MongoDB](https://www.mongodb.com/) v2.4+ as its database.


## Install Node.js (Mac OS)

This is how to install node on Mac OS using [homebrew](http://brew.sh/). For other operating systems, please refer to the [Node.js download page](https://nodejs.org/en/download/).

```shell
brew update
brew install node
```

## Setup local MongoDB (Mac OS)

This is the installation guide to get a local development installation on Mac OS using [homebrew](http://brew.sh/). For other operating systems, please refer to the [MongoDB Getting Started Guide](https://docs.mongodb.com/getting-started/shell/).

```shell
brew update
brew install mongodb
mongod --config /usr/local/etc/mongod.conf
```

Now the mongo daemon should be running in the background. To have mongo start automatically as a background service on startup you can also do:

```shell
brew services start mongodb
```

Now you can use the `mongo` CLI client to create a new test database. **The username and password used here match the ones in the `credentials.json` file. Be sure to change them for production use**:

```shell
mongo
use keyserver-test
db.createUser({ user:"keyserver-user", pwd:"trfepCpjhVrqgpXFWsEF", roles:[{ role:"readWrite", db:"keyserver-test" }] })
```

## Setup SMTP user

The key server uses [nodemailer](https://nodemailer.com) to send out emails upon public key upload to verify email address ownership. To test this feature locally, open the `credentials.json` file and change the `smtp.user` and `smtp.pass` attributes to your Gmail test account. Make sure that `smtp.user` and `sender.email` match. Otherwise the Gmail SMTP server will block any emails you try to send. Also, make sure to enable `Allow less secure apps` in the [Gmail security settings](https://myaccount.google.com/security#connectedapps). You can read more on this in the [Nodemailer documentation](https://nodemailer.com/using-gmail/).

For production you should use a service like [Amazon SES](https://aws.amazon.com/ses/), [Mailgun](https://www.mailgun.com/) or [Sendgrid](https://sendgrid.com/solutions/transactional-email/). Nodemailer supports all of these out of the box.

## Install dependencies and run tests

```shell
npm install && npm test
```

## Start local server

```shell
npm start
```



# Production

The `credentials.json` file can be used to configure a local development installation. For production use, the following environment variables need to be set:

* NODE_ENV=production
* MONGO_URI=127.0.0.1:27017/test_db
* MONGO_USER=db_user
* MONGO_PASS=db_password
* SMTP_HOST=127.0.0.1
* SMTP_PORT=465
* SMTP_TLS=true
* SMTP_USER=smtp_user
* SMTP_PASS=smtp_pass
* SENDER_NAME="OpenPGP Key Server"
* SENDER_EMAIL=noreply@example.com



# License

AGPL v3.0

See the [LICENSE](https://raw.githubusercontent.com/mailvelope/keyserver/master/LICENSE) file for details

## Libraries

Among others, this project relies on the following open source libraries:

* [OpenPGP.js](https://openpgpjs.org/)
* [Nodemailer](https://nodemailer.com/)
* [addressparser](https://github.com/nodemailer/addressparser)
* [koa](http://koajs.com/)
* [mongodb](https://mongodb.github.io/node-mongodb-native/)
