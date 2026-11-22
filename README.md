## fuites anarchive v2

fuit.es aims to provide a creative online space, a platform for collective processes designed with the ‘Anarchive’ as its operational concept; an archive which promotes the continual evolution and variation of its documents rather than their mere preservation.

Each individual page offers a space for collaboration, where digital media in the form of images, sounds or text can be uploaded and composed with. These individual media objects are referred to as “thingies”. Anyone with a token can create a verse, a public network of pages for anarchiving thingies. If you are curious about accessing the platform, get in touch with us by email at contact@fuit.es, we would be happy to make you a token.

## Repository

The repo uses npm workspaces. Packages are divided up into a backend application (be) and frontend (fe) packages. To install and run locally:

1. clone or fork the repository:
2. Make sure to have installed and be using Node.js 20.9 or higher.
3. Configure a self-signed certificate for local HTTPS support (see the section below on how to do this).
4. Add .env files to both the `packages/be` directory and the `packages/fe` directory (email contact@fuit.es to get these files)
5. Open an SSH tunnel to the database (see the section below on how to do this).
6. from the repo root run `npm ci` and then run `npm run be:dev`
7. In a second terminal (or command line) window, from the repo root run `npm run fe:dev`


## Database access

MongoDB runs on the droplet and listens on `127.0.0.1` only, so it accepts connections from the droplet itself and is not reachable over the internet. Local development reaches the development database through an SSH tunnel.

Add the following to `~/.ssh/config`, filling in the host, port and user from the credentials sent with the .env files:

    Host fuites-db-tunnel
      HostName <droplet-ip>
      Port <ssh-port>
      User <ssh-user>
      LocalForward 27017 127.0.0.1:27017
      ExitOnForwardFailure yes
      ServerAliveInterval 60

Then run `ssh -N fuites-db-tunnel` in its own terminal window and leave it open while developing. `DATABASE_URL` in `packages/be/.env` points at `127.0.0.1:27017`, which the tunnel forwards to the droplet. A `MongooseServerSelectionError: connect ECONNREFUSED` on startup usually means the tunnel is not running.


## Self-signed certificate

In order to use the site in local development mode, two files must be added to the repo root. 
The following commands will work on MacOS but if mkcert is installed by another package manager, this can be run on any version of *nix.

In a terminal shell:
1. `cd ~/.ssh`
2. `brew install mkcert`
3. `mkcert -install`
4. `mkcert -key-file localhost_key.pem -cert-file localhost_cert.pem localhost 127.0.0.1`
5. `cat localhost_cert.pem > localhost_fullchain.pem`
6. `cat "$(mkcert -CAROOT)/rootCA.pem" >> localhost_fullchain.pem`

Now, navigate back to your cloned repo, into the root directory. Add the two generated .pem files to the repo with the following command. These keys are .gitignored by default.

`cp -v ~/.ssh/localhost_cert.pem ~/.ssh/localhost_key.pem .`
