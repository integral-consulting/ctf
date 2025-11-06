# ctf

Dette prosjektet dokumenterer hvordan man setter opp [CTFd][ctfd] som er lisensiert under Apache License 2.0.

## Forutsetninger

* En virtuell maskin (VM) med et OS som har støtte for docker
  * I dette eksemplet er det benyttet [Ubuntu Server 24.04 LTS][ubuntu-server]
* Python + Ansible (valgfritt)
  * Anbefalt at du har SSH-tilgang til VM'en hvis du ønsker å benytte ansible playbooken

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

For å benytte ansible, oppdater [hosts][hosts] med hostname, ip og brukernavn på maskinen som skal konfigureres.
Deretter kjør:

```shell
ansible-playbook -i hosts config.yml --ask-become-pass
```

Dette vil installere docker på tilsvarende måte som i [get started][docker-get-started] guiden, samt opprette og tilordne gruppe til brukeren. Restart maskinen eller kjører `newgrp docker` for å kunne utføre `docker` kommandoer uten å benytte sudo.

## Lisens og kilde

All dokumentasjon og konfigurasjon i dette repoet er også delt under 
[Apache License 2.0](LICENSE).


[ctfd]: https://ctfd.io/
[ubuntu-server]: https://ubuntu.com/download/server
[docker-get-started]: https://docs.docker.com/engine/install/ubuntu/
[hosts]: ./hosts
[config]: ./config.yml
