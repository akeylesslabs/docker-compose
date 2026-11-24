# Modern PAM with an Akeyless Gateway

An example Docker Compose for evaluating Modern PAM on one host: the Akeyless Gateway, the Modern PAM
bastion (SSH, browser SSH, database and remote desktop sessions) and the remote desktop engine.
**Modern PAM is in public beta.** This stack is for evaluation and small pilots, not production. For production use
the unified `akeyless-gateway` Helm chart with `modernPam.enabled: true`.

It is separate from the V1 files in the folder above. A gateway sends session setup to one bastion address per
route, so a gateway serves the V1 bastion or the Modern PAM one and not both. Do not point one gateway at both.

## Before you start

- Docker and Docker Compose v2.
- A gateway identity: an access id and key the gateway signs in with, on an Akeyless account.
- A separate API key identity for Modern PAM itself (the broker), with permission to read the items it serves.
- The gateway's SSH certificate issuer public key, one line. With the Akeyless CLI:
  `akeyless get-rsa-public --name <your ssh cert issuer's key> | awk -F'- SSH: ' '/- SSH:/{print $2}'`
- The host key of each SSH target: `ssh-keyscan -p <port> <host>`.

## Set it up

1. Edit `gateway.env` and `sra.env` and replace every `XXXXXX`.
2. Create the bastion's host key, once. It must stay the same across restarts:
   `ssh-keygen -t rsa -b 2048 -m PEM -f ./hostkey -N '' -q && chmod 644 ./hostkey`
3. Put your targets' host keys in `./known_hosts`, one line each. A target that is not listed is refused.
4. Start it with a published `akeyless/modern-pam` tag:
   `MODERN_PAM_VERSION=<tag> docker compose up -d`

The bastion starts after the gateway reports healthy. If you start the services yourself in another order, the
bastion restarts until the gateway answers, which is harmless.

## Check it

- Gateway: `curl http://localhost:8000/health` answers `Health Check Ok`.
- Bastion: `docker logs modern-pam` shows `preflight check ... GW-AUTH ... PASS` and `ssh gate listening`.
- Connect: `akeyless connect -n <item> -t <user>@<target>:<port> -g http://<host>:8000 --via-sra <host>:2222`.
  The gateway lists the bastion for the CLI to pick only on a gateway release that carries bastion registration
  (5.8.0 and later), so on an older one name it with `--via-sra`.

## Notes

- The first connection from a client asks about the bastion's host key. It does not change while `./hostkey` stays.
- `GATEWAY_VERSION` selects the gateway image tag, `latest` by default.
- Moving from the V1 stack: see the migration guide of the Modern PAM documentation, and stop the V1 stack first.
