# The Bannion daemon

The daemon lets Bannion reach the systems inside your own network. You run it on a machine in that network. It makes
one connection out to Bannion on port 443. Nothing has to reach in, so you open no ports.

```
ghcr.io/bannion-labs/bannion-daemon
```

It runs on Linux, on both amd64 and arm64. Docker picks the right one for your machine.

## Start in Bannion

Set the daemon up from Bannion, not from this page. In Bannion, go to **Connections** and add your own network. Bannion
gives you a token, a snippet for the config file, and the exact command to run. That command names this image by its
digest, so you get exactly the image Bannion lists.

## Run it

1. Make a folder for the daemon.
2. Put the snippet from Bannion in a file called `tunnel-daemon.properties` in that folder.
3. Run the command Bannion shows you from inside that folder. On macOS and Linux it looks like this:

   ```
   docker run -d --name bannion-daemon --restart unless-stopped \
     --user "$(id -u):$(id -g)" \
     --add-host=host.docker.internal:host-gateway \
     -v "$PWD":/etc/bannion \
     ghcr.io/bannion-labs/bannion-daemon@sha256:<the digest Bannion shows you> run
   ```

   On Windows, Docker Desktop has to be running. Open PowerShell in the folder, then run this:

   ```
   docker run -d --name bannion-daemon --restart unless-stopped `
     --add-host=host.docker.internal:host-gateway `
     -v "${PWD}:/etc/bannion" `
     ghcr.io/bannion-labs/bannion-daemon@sha256:<the digest Bannion shows you> run
   ```

   If you make the file in Notepad, set *Save as type* to *All files*. Otherwise it saves as
   `tunnel-daemon.properties.txt` and the daemon cannot find it.

4. Go back to Bannion and press **Check**. Bannion tells you when the daemon has reached it.

**Keep the folder.** The daemon keeps its key in that folder, next to your config. Always start it with the folder
mounted like this. If it starts without the folder, it cannot get back in, and you will need a new token from Bannion.

## Tell it which servers to reach

Add one line to `tunnel-daemon.properties` for each server, with the server's full address:

```
targets.inventory = https://inventory.internal.example:8443/api
```

The daemon sees the change in a few seconds. You do not have to restart it.

Inside the daemon, `localhost` means the daemon itself, not your machine. For a server on the same machine, use
`host.docker.internal` in its address. On Linux, that server also has to listen on more than `127.0.0.1`. Or add
`--network host` to the command and keep `localhost`.

## Check it before you leave it running

From the folder that holds the config:

```
docker run --rm --user "$(id -u):$(id -g)" -v "$PWD":/etc/bannion \
  ghcr.io/bannion-labs/bannion-daemon@sha256:<digest> check
```

It connects for real and tells you what worked. It needs the folder mounted, like `run` does.

## What your network has to allow

All of these go out on port 443:

- `tunnel.bannion.ai`, where the daemon connects to Bannion
- `ghcr.io` and `pkg-containers.githubusercontent.com`, to download this image

## Servers with their own certificates

**A certificate your company signs.** Save the certificate your company signs with as `ca.pem` in the daemon's folder.
Then add this line to `tunnel-daemon.properties`:

```
trust.pem.path = ca.pem
```

The daemon still trusts the usual public certificates too.

**A certificate the server made itself.** Run `check`. For each server it prints the fingerprint of the certificate it
was sent, and the exact line to add:

```
target-tls.inventory.pin = sha256:3fa1…9c
```

Copy that line into `tunnel-daemon.properties`. The daemon then accepts exactly that certificate for that server. When
the server gets a new certificate, run `check` again and copy the new line.

## Check the signature

Every image is signed with Bannion's own key. In Bannion, the Daemon step has a section called **Other ways to run it**.
It shows the exact command to check the signature, with the address of the key. It looks like this:

```
cosign verify --key <the key's address from Bannion> --insecure-ignore-tlog=true \
  ghcr.io/bannion-labs/bannion-daemon@sha256:<digest>
```

Bannion does not write its signatures to the public Sigstore log, so the check is the key alone. That is why the
command needs `--insecure-ignore-tlog=true`.

## Other ways to run it

- **Podman and Colima** run the same command. With Podman, add `--userns=keep-id`. Colima only sees folders inside your
  home folder, so make the daemon's folder there.
- **Kubernetes.** Put the config in a Secret. Keep the key on a volume that outlives the pod, and point
  `node.identity.path` at a file on it.

## What is inside

- One program, `/bannion-daemon`, built ahead of time. There is no shell and no Java runtime in the image.
- It is built on Chainguard's `glibc-dynamic` image.
- It runs as user `65532` unless you pass `--user`.
- Its working folder is `/etc/bannion`, the folder you mount.

## Tags

Use the digest Bannion shows you. Each release also has a tag, like `0.0.1-latest`, but a tag can be moved to another
image and a digest cannot.
