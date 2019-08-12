# kadira-compose

This will get [open-source Kadira](https://github.com/kadira-open/kadira-server)
running on a dedicated host, almost fully automatically using `docker-compose`.
Specifically, it includes the following dockers:

* [MongoDB server](https://hub.docker.com/_/mongo/):
  storing data in `/opt/kadira-mongo`, and accessible via port 27017
* [nginx-proxy](https://github.com/jwilder/nginx-proxy): maps ports 80, 443 (SSL), and 22022 (SSL) to kadira-engine's 11011
* [letsencrypt-nginx-proxy-companion](https://github.com/JrCs/docker-letsencrypt-nginx-proxy-companion):
  Acquires Let's Encrypt SSL certificate to support https access.
* kadira-engine: This is where Meteor apps send their data.
  Accessible via `http://HOST:11011` or `https://HOST:22022`.
* kadira-rma: This computes statistics every minute as needed by kadira-ui.
* kadira-ui: Meteor app to present Kadira user interface,
  accessible via `http://HOST:4000`.

The Dockerification of open-source Kadira was originally done by
[Vlad Holubiev](https://github.com/vladgolubev/), as described in
[this Meteor forum post](https://forums.meteor.com/t/running-a-own-kadira-instance-update-now-with-a-guide/35938/34).
What I've added here is:
* `https` access to kadira-engine (via nginx-proxy)
* built-in Mongo server instead of relying on an external service like mlab.com
* cron job to clear out old database entries (see below)

Unfortunately, because of the way letsencrypt-nginx-proxy-companion works,
I need ports 80 and 443 to point to kadira-engine instead of kadira-ui.
So you need to access the UI via http port 4000.

## Configuration

* **Expose** the following ports on your host:
  * 22 (if you want to ssh in)
  * 80 &amp; 443 (for letsencrypt to work)
  * 4000 (for Kadira UI)
  * 11011 (if your Meteor app uses `http`)
  * 22022 (if your Meteor app uses `https`)

* **Do not expose** port 27017, or else your MongoDB will be vulnerable.

* [Install **Docker**](https://docs.docker.com/engine/installation/)

* Install/download [**docker-compose**](https://docs.docker.com/compose/install/#install-compose)

* **Clone** this repo.

* **Edit** `.env` to change `HOST` and `EMAIL` environment variables.

* If you want to **clear out database of old entries**
  (as [recommended here](https://forums.meteor.com/t/running-a-own-kadira-instance-update-now-with-a-guide/35938/170)
  for long-term performance), add the included `crontab` to your existing
  crontab, or if you don't have one, just run `crontab crontab` (on Linux).
  You may need to edit the path to the included `flush-mongo` script.

## Running

* Launch dockers, and initialize Mongo replica set:

  ```
  docker-compose up -d mongo
  docker-compose exec mongo mongo --eval 'rs.initiate({_id:"kadira", members: [{_id: 0, host: "mongo:27017"}]})'
  docker-compose up -d
  ```

* Open `http://HOST:4000` and login with email `admin@gmail.com`
  (initial password `admin` -- change it immediately in the UI)

* Create your apps

* Upgrade apps to business plan:

  ```
  docker-compose exec mongo mongo kadira --eval 'db.apps.update({},{$set:{plan:"business"}},{multi:true})'
  ```

* Visit `http://HOST:4000` to monitor your apps.

## Meteor Setup

If your server is running on `https`, use the port-22022 endpoint
to kadira-engine.  For example, add the following line to `server/kadira.js`:

```javascript
Kadira.connect(APP_ID, APP_SECRET, {endpoint: 'https://HOST:22022'})
```

If your server is running on `http`, use the port-11011 endpoint:

```javascript
Kadira.connect(APP_ID, APP_SECRET, {endpoint: 'http://HOST:11011'})
```
