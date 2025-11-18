# ctf

Dette repoet dokumenterer hvordan man setter opp [CTFd][ctfd] på 1 VM med docker installert. Oppsettet egner seg best for små team og kan brukes som et startpunkt for å raskt komme i gang med en CTF. 

> [!CAUTION]
> Dette oppsettet er å anse som en Proof-of-Concept på hvordan man kan self-hoste [CTFd][ctfd] og inneholder noen "snarveier" til mål som går på bekostning av gode sikkerhetsprinsipper. Disse snarveiene er markert i instruksjoene. Det anbefales derfor å sette dette opp i eget lukket miljø, med mindre man har utbedret snarveiene.

> [!NOTE]
> Er planen å gjennomføre en større CTF med mange deltakere bør man vurdere å se på hvordan løsningen kan deployes i en orkestreringsplattform som kubernetes eller docker swarm. Se [ctfd-deploybot] som tar utgangspunktet i docker swarm for dette.

## Forutsetninger

* En virtuell maskin (VM) med et OS som har støtte for docker
  * I dette eksemplet er det benyttet [Ubuntu Server 24.04 LTS][ubuntu-server]
* Python + Ansible (valgfritt)
  * Anbefalt at du har SSH-tilgang til VM'en hvis du ønsker å benytte ansible-playbooken

### Installasjon av ansible (valgfritt)

Hvis du ikke vil benytte ansible kan du hoppe rett til [Konfigurasjon av VM](#konfigurasjon-av-vm).

#### Opprett virtual environment 

Enkleste måten å installere ansible på er ved å benytte python og et virtual environment:

```shell
# Opprett virtual environment the good old plain python way:
python -m venv .venv

# Eventuelt benytt uv
uv venv
```

Aktiver miljøet med:

```shell
source .venv/bin/activate
```

#### Installer ansible

Nå kan vi installere ansible med 

```python
pip install ansible
```

## Konfigurasjon av VM

For å hoste [CTFd][ctfd] selv trenger vi docker og docker compose.
Dette kan installeres med ansible playbook'en [config.yml][config] i repoet, eller ved å følge docker sin [get started][docker-get-started] guide.

For å benytte ansible, oppdater [hosts] med hostname, ip og brukernavn på maskinen som skal konfigureres.
Deretter kjør:

```shell
# --ask-become-pass prompter etter sudo passord på maskinen du skal konfigurere
ansible-playbook -i hosts config.yml --ask-become-pass
```

Dette vil installere docker på tilsvarende måte som i [get started][docker-get-started] guiden, samt opprette og tilordne gruppe til brukeren. Restart maskinen eller kjører `newgrp docker` for å kunne utføre `docker` kommandoer uten å benytte sudo.

## Installasjon av CTFd

Herfra og utover vil alle kommandoer utføres på maskinen hvor CTFd skal kjøre - altså VM'en vi konfigurerte i forrige steg. Disse instruksjonene er basert på [ctfd-whale] sin fremgangsmåte for å installere plattformen inkludert whale pluginen. Whale pluginen er en plugin til CTFd-plattformen for å kunne la plattformen kjøre opp containere dynamisk på oppgaver som krever dette. 

Det første vi gjør er å enable docker swarm og sette label på noden:

```shell
# Enable docker swarm
docker swarm init
# Label node
docker node update --label-add "name=linux-1" $(docker node ls -q)
```

Deretter cloner vi ned ctfd repoet og navigerer til dette:

```shell
git clone https://github.com/ctfd/ctfd && cd ctfd
```

Dette repoet inneholder `docker-compose.yml` som vi må oppdatere for at [whale-pluginen][ctfd-whale] skal fungere.
Legg derfor til følgende under `services:`

```yml
  frps:
    image: glzjin/frp
    restart: always
    volumes:
      - ./conf/frp:/conf
    entrypoint:
      - /usr/local/bin/frps
      - -c
      - /conf/frps.ini
    ports:
      - 10000-10100:10000-10100  # for "direct" challenges
      - 8001:8001  # for "http" challenges
    networks:
      default:  # frps ports should be mapped to host
      frp_connect:

  frpc:
    image: glzjin/frp:latest
    restart: always
    volumes:
      - ./conf/frp:/conf/
    entrypoint:
      - /usr/local/bin/frpc
      - -c
      - /conf/frpc.ini
    depends_on:
      - frps #need frps to run first
    networks:
      frp_containers:
      frp_connect:
        ipv4_address: 172.1.0.3
``` 

I tillegg må vi legge til følgende under `networks`:

```yml
      frp_connect:
      driver: bridge
      internal: true
      attachable: true
      ipam:
        config:
          - subnet: 172.1.0.0/16
    frp_containers:
      driver: overlay
      internal: true
      attachable: true
      ipam:
        config:
          - subnet: 172.2.0.0/16
```

Deretter må vi opprette en mappe med konfigurasjon for frps(fast reverse proxy):

```shell
mkdir -p conf/frps
```

Her trenger vi 2 filer:

frps.ini med følgende innhold:

```ini
[common]
# following ports must not overlap with "direct" port range defined in the compose file
bind_port = 7987  # port for frpc to connect to
vhost_http_port = 8001  # port for mapping http challenges
token = your_token
subdomain_host = ctf.example.com # hostname that's mapped to frps by some reverse proxy (or IS frps itself)
```

og frpc.ini

```
[common]
token = your_token
server_addr = frps
server_port = 7897  # == frps.bind_port
admin_addr = 172.1.0.3  # refer to "Security"
admin_port = 7400
```

> ![CAUTION]
> Det er ikke anbefalt å eksponere docker.socket'en til kjørende containere. Dette bør man finne andre løsninger på hvis man setter opp løsningen på en åpen plattform.

Videre trenger ctfd plattformen tilgang på docker socket'en på hosten for å kunne spinne opp containere når brukere skal starte oppgaver som er avhengig av andre docker containere.
Dette gjør vi gjennom å oppdatere `docker-compose.yml` med følgende verdier under `services: ctfd:`:

```yml
services:
    ctfd:
        ...
        volumes:
            - /var/run/docker.sock:/var/run/docker.sock
        depends_on:
            - frpc #need frpc to run ahead
        networks:
            ...
            frp_connect:
```

Det som gjenstår nå er å legge til [ctfd-whale]-pluginen, dette gjør vi ved å clone ned deres repo inn i plugin folderen til CTFd på følgende måte:

```shell
git clone https://github.com/frankli0324/CTFd-Whale CTFd/plugins/ctfd-whale --depth=1
```

Nå skal vi være klar til å kjøre opp løsningen:

```shell
# For eldre docker
docker-compose build
docker-compose up -d

# For nyere docker
docker compose build
docker compose up -d
```

Nå skal løsningen kjøre og være tilgjengelig på http://localhost:8080 (forutsatt at man sitter på samme nettverk som serveren benyttet til å kjøre løsningen).

Deretter kan man sette opp CTF arrangementet som beskrevet i [ctfd sin getting started][ctfd-getting-started].

## Lisens og kilde

All dokumentasjon og konfigurasjon i dette repoet er også delt under [Apache License 2.0](LICENSE) slik som [ctfd]-løsningen også er.


[ctfd]: https://ctfd.io/
[ubuntu-server]: https://ubuntu.com/download/server
[docker-get-started]: https://docs.docker.com/engine/install/ubuntu/
[hosts]: ./hosts
[config]: ./config.yml
[ctfd-whale]: https://github.com/frankli0324/ctfd-whale/blob/master/docs/install.md
[ctfd-deploybot]: https://docs.ctfd.io/enterprise/deploybot/cluster/
[ctfd-getting-started]: https://docs.ctfd.io/tutorials/getting-started/
