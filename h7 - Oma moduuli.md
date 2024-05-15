## Oma Moduuli - Jonas Julin

Tämän moduulin tarkoituksena on asentaa hyödyllisiä ohjelmia nopeasti.

Käytin Vagrantia jolla tein master ja minion koneen. Vagrant pyöri Windows 10 koneella, jossa 16 GB RAM ja Ryzen 5 3600-prosessori.
Master ja minion koneet ovat Debian/Bullseye64 koneita.

## Lisenssi

GNU General Public License, version 3.
https://github.com/jonzsa92/palvelinhallinta/tree/main?tab=GPL-3.0-1-ov-file#readme

## Vagrant, master and minion.

Vagrant oli minulla jo asennettuna koneella, mutta sen saa helposti asennettua Windowsille heidän omalta sivultaan: `https://developer.hashicorp.com/vagrant/downloads`, ja sieltä valitsee oman käyttöjärjestelmän. En käy kummemin Vagrantin asennusta tässä.

Tein koneelleni uuden kansion, jonka nimeksi tuli nyt "Omamoduuli".

Avasin PowerShellin admin-käyttäjällä, siirsin näkymän komenolla `cd C:\Users\Jonzsa\Desktop\Omamoduuli` , ja siellä käytin komentoa `New-Item -Path "C:\Users\Jonzsa\Desktop\Omamoduuli\Vagrantfile" -ItemType File`. Näin saatiin tyhjä tiedosto tehtyä.

Seuraavaksi komentona `notepad.exe Vagrantfile` jotta saan muokattua tiedostoa ilman että se muuttaa tyyppiään.

Käytin Tero Karvisen ohjetta, pienillä muokkauksilla, tehdäkseni kaksi konetta samassa verkossa. 
    
    https://terokarvinen.com/2021/two-machine-virtual-network-with-debian-11-bullseye-and-vagrant/?fromSearch=vagrant%20two%20machine

Tässä Vagrantfile-tiedoston sisältö:

    # -*- mode: ruby -*-
    # vi: set ft=ruby :
    # Copyright 2019-2021 Tero Karvinen http://TeroKarvinen.com

    $tscript = <<TSCRIPT
    set -o verbose
    apt-get update
    apt-get -y install tree
    echo "Done - set up test environment - https://terokarvinen.com/search/?q=vagrant"
    TSCRIPT

    Vagrant.configure("2") do |config|
	    config.vm.synced_folder ".", "/vagrant", disabled: true
	    config.vm.synced_folder "shared/", "/home/vagrant/shared", create: true
	    config.vm.provision "shell", inline: $tscript
	    config.vm.box = "debian/bullseye64"

	    config.vm.define "master" do |master|
		    master.vm.hostname = "master"
		    master.vm.network "private_network", ip: "192.168.88.101"
	    end

	    config.vm.define "minion", primary: true do |minion|
		    minion.vm.hostname = "minion"
		    minion.vm.network "private_network", ip: "192.168.88.102"
	    end
	
    end

Tämän jälkeen, komento (edelleen samassa kansiossa kuin mihin luotiin Vagrantfile), `vagrant up`. Tarkistetaan että päästään koneisiin käsiksi:

    vagrant ssh master
    exit
    vagrant ssh minion
    exit

Pingataan vielä koneesta koneelle:

    vagrant ssh master
    ping -c 1 192.168.88.102
    exit

    vagrant ssh minion
    ping -c 1 192.168.88.101
    exit

Sitten tehdään master masteriksi, ja minion minioniksi.

Master:

    sudo apt-get install -y salt-master

Minion:

    sudo apt-get install -y salt-minion

Minion joutuu vielä muokkaamaan konfiguraatiota että se liittyy masterin alle:

    sudoedit /etc/salt/minion //<- tuonne lisätään vain master: 192.168.88.101 (master ip-osoite)
    sudo systemctl restart salt-minion

Takaisin master koneelle, joudutaan hyväksymään minion avain, komento `sudo salt-key -A`. Y for yes.

## Moduuli

Halusin asentaa muutaman ohjelman koneelle, ja näyttää että se menee helposti. Tässä moduulissa asennetaan FireFox, Apache2, Git, Curl, Micro, SSH ja palomuuri. Vagrantilla ssh yhteys masterkoneeseen `vagrant ssh master`. 

Seuraavaksi tehdään kansio, `sudo mkdir /srv/salt`. Tähän kansioon tuodaan kaikki mitä halutaan asentaa. Tehdään jo kansiot niille, jota halutaan asentaa.

    cd /srv/salt
    sudo mkdir firefox apache2 git curl micro ssh ufw

Tämän jälkeen voidaan tehdä jo valmiiksi top.sls tiedosto:

    sudoedit top.sls

Tässä on top.sls:n tiedot:

![image](https://github.com/jonzsa92/palvelinhallinta/assets/106398186/5c0ad0ab-c640-4fa3-b93b-6751e777d362)



Tämän jälkeen, tehdään jokaiselle alikansiolle oma init.sls.

Eli jokainen kansio kun olet /srv/salt kansiossa:

    cd ./[alikansion nimi]
    sudoedit init.sls // laita sen alikansion ohjelman init.sls tiedon sisältö tähän
    cd ..

Rinse & repeat jokaiselle jonka teet.

Listaan tähän alle kaikkien init.sls:ien tiedot

    Firefox
    firefox-esr:
      pkg.installed

    Apache2
    apache2:
      pkg.installed

    apache2 Service:
      service.running:
        - name: apache2

    Curl
    curl:
      pkg.installed

    Micro
    micro:
      pkg.installed

    Git
    git:
      pkg.installed

    SSH
    ssh:
      pkg.installed:
        - pkgs:
          - openssh-client
          - openssh-server

    UFW
    ufw:
      pkg.installed

Ajetaan sen jälkeen `sudo salt '*' state.apply`

Ja kaikki asennettu

![image](https://github.com/jonzsa92/palvelinhallinta/assets/106398186/ae625aa0-4e1c-460b-b1d7-ee4c6afcaed7)


Ajattelin että olisin myös muokannut Apache2:n localhost sivun omaksi, ja firefoxin aloitussivun googleksi, mutta meni vähän jännäksi ja aika alkoi loppumaan.

## Lähteet:

Tero Karvinen - Two Machine Virtual Network With Debian 11 Bullseye and Vagrant -

https://terokarvinen.com/2021/two-machine-virtual-network-with-debian-11-bullseye-and-vagrant/?fromSearch=vagrant%20two%20machine
