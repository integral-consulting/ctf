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

```shell
git clone https://github.com/ctfd/ctfd
```

## Lisens og kilde

All dokumentasjon og konfigurasjon i dette repoet er også delt under [Apache License 2.0](LICENSE) slik som [ctfd]-løsningen også er.


[ctfd]: https://ctfd.io/
[ubuntu-server]: https://ubuntu.com/download/server
[docker-get-started]: https://docs.docker.com/engine/install/ubuntu/
[hosts]: ./hosts
[config]: ./config.yml
[ctfd-whale]: https://github.com/frankli0324/ctfd-whale/blob/master/docs/install.md
[ctfd-deploybot]: https://docs.ctfd.io/enterprise/deploybot/cluster/
