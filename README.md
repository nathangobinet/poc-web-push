# Web Push POC

Proof of concept for regex log notification subscribtion using the web push technology.

## How it works

### Client side

First user must allow notifications. Some browser, like safari, require user interaction to prompt the ask so a dedicated button is used. While notifications are not allowed by client nothing is done.

Notification reception is handled by a service worker. Service worker is a dedicated file that run in the browser background. It updates itself automatically each 24h. It use a dedicated scope that allows specific usage (https://developer.mozilla.org/en-US/docs/Web/API/ServiceWorkerGlobalScope). It define 2 events handler, one when notification is received, and the other when the notifications is clicked (see code).

A service worker is registered if it doesnt exist on app startup.
Notification subscription are linked to the service worker. If a  service worker is unregistred, all existing subscription and notififcation are deleted.

Once service worker is retrieved or created, an event is attached to it to be able to receive message from it on a new notification.

The subscription is then retrieved or created from the service worker. A subscription is an object with an endpoint to notify and auth keys. It need a public VAPID key (from the server) to be created.

Once evrything is set (serviceworker and subscription), the client must symply send it subscription ton the server in order to be notifyed in the future.

### Server side

Notifications are send using [HTTP::Request::Webpush](https://metacpan.org/pod/HTTP::Request::Webpush). Server must first store subscription objects (from clients) and then can use Webpush to send notification to the wanted subscriptions. In order to encode the payload and provide authentification header it must have a private and and public (the same used in subscription creation) VAPID keys.

Subscriptions are stored in a SQLite Database following this schema :

```sql
CREATE TABLE IF NOT EXISTS alerts (
    id INTEGER PRIMARY KEY,
    group_name TEXT NOT NULL,
    name TEXT NOT NULL,
    regex TEXT NOT NULL
);

CREATE TABLE IF NOT EXISTS subscriptions (
    id INTEGER PRIMARY KEY,
    browser_info JSON NOT NULL
);

CREATE TABLE IF NOT EXISTS alerts_subscriptions (
    alert_id INTEGER NOT NULL,
    subscription_id INTEGER NOT NULL,
    PRIMARY KEY (alert_id, subscription_id),
    FOREIGN KEY (alert_id) REFERENCES alerts(id) ON DELETE CASCADE,
    FOREIGN KEY (subscription_id) REFERENCES subscriptions(id) ON DELETE CASCADE
);
```

In order to make `ON DELETE CASCADE` works it's necessary to specify:

```sql
PRAGMA foreign_keys = ON;
```

## Installation

### Install Perl dependencies

```shell
# New
cpanm HTTP::Request::Webpush
# Should already be installed
cpanm JSON::XS;
cpanm Mojolicious::Lite;
# Optional
cpanm Dotenv;
```

### Generate Public & Private VAPID keys

- Using openssl

  ```
  TODO
  ```

- Using npx

  ```
  npx web-push generate-vapid-keys
  ```

- Using browser 

  https://www.stephane-quantin.com/en/tools/generators/vapid-keys

### Set env

Public key must be share with front (api incoming).
For the server there is 2 ways :

- With .env file

  ```conf
  PRIVATE_KEY=...
  PUBLIC_KEY=...
  ```

  Dotenv should be use and loaded in perl

  ```pl
  use Dotenv -load;
  ```

- With env profile

  ```shell
  echo export PRIVATE_KEY=... >> ~/.profile
  echo export PUBLIC_KEY=... >> ~/.profile
  ```

### Setup the database

Create `web-push-server.db` and use the schema.
