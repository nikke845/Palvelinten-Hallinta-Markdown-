#Palvelinten hallinta viikko3

Tätä dokumenttia saa kopioida ja muokata GNU General Public License (versio 2 tai uudempi) mukaisesti. [http://www.gnu.org/licenses/gpl-3.0.html](http://www.gnu.org/licenses/gpl-3.0.html)
Pohjana Tero Karvinen 2020: [Palvelinten Hallinta – Spring 2020](http://terokarvinen.com/2020/configuration-managment-systems-palvelinten-hallinta-ict4tn022-spring-2020/)

Tehtävän teko aloitettu 21.4.2020 noin klo 17.00.

##Uusi Salt moduuli

Moduuli on asennettu [tämän ohjeen mukaan.](https://itsfoss.com/disable-touchpad-when-mouse-used/)

Asennetaan touchpad-indicator koneeseen, jotta hiiren liittäminen disabloisi automaattisesti touchpadin läppäristä.

Ensin manuaalinen asennus:

	sudo add-apt-repository ppa:atareao/atareao
	sudo apt update
	sudo apt install touchpad-indicator

Etsitään juuri muuttuneet tiedostot:

	find /etc/ $HOME -printf '%T+ %p\n'|sort|grep touch

```
2020-04-21+17:13:47.0497477300 /home/nikke/.config/touchpad-indicator
2020-04-21+17:13:48.2257608270 /home/nikke/.cache/gnome-software/icons/6f7c2a42b0dbda80476e36514450bd8fe42ce0b1-remote-touchpad_128.png
2020-04-21+17:13:50.5777868600 /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf
```

Kokeilin myös hakea muualta conffi filuja, mutta en löytänyt esimerkiksi kaikkien käyttäjien yhteistä conffia.
Seuraavaksi kokeilin muuttaa graafisesta käyttöliittymästä asetuksen päälle, joka poistaa touchpadin käytöstä, kun hiiri on kytkettynä koneeseen.
Kokeillaan löytää muutos:
	find /etc/ $HOME -printf '%T+ %p\n'|sort| grep touch

Muutettu tiedosto sijaitsee siis käyttäjän kotihakemistossa.
	2020-04-21+17:33:16.0351815510 /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf

Koska ohjelman .conf tiedosto on vain käyttäjän kotihakemistossa ja kaikki käyttäjät eivät luultavasti halua samaa asetustiedostoa teen tämän conffin vain yhtä käyttäjää ajatellen.
Mennään .conf tiedoston sijaintiin ja siirretään se palvelimelle:
	cd /home/nikke/.config/touchpad-indicator
	rsync /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf <käyttäjä>@<domain/ip>:/home/<käyttäjä>/
Siirrytään palvelimelle, johon .conf tiedosto juuri siirrettiin:
	ssh <käyttäjä>@<domain/ip>
Varmistetaan, että tiedosto löytyy
	cat touchpad-indicator.conf
Tehdään touchpad-indicator -tilalle kansio:
	sudo mkdir /srv/salt/touchpad-indicator
Viedään kotihakemistossa oleva .conf tiedosto kansioon:
	sudo cp /home/<user>/touchpad-indicator.conf /srv/salt/touchpad-indicator/
Tehdään tilalle sls tiedosto:
	sudoedit /srv/salt/touchpad-indicator/touchpad-indicator.sls
Johon laitetaan seuraavaa:

```
base:
  pkgrepo.managed:
    - name: ppa:atareao/atareao
    - dist: precise
    - file: /etc/apt/sources.list.d/atareao.list

touchpad-indicator:
  pkg.installed:
    - fromrepo: ppa:atareao/atareao

/home/nikke/.config/touchpad-indicator/touchpad-indicator.conf:
  file.managed:
    - source: salt://touchpad-indicator/touchpad-indicator.conf
    - user: nikke
    - group: nikke
    - mode: 664
```

Lisätään touchpad-indicator -tila top.sls tiedostoon ensiksi laitteelle, jolle se on jo asennettuna:
	base:
	  'hp1020g1':
	    - touchpad-indicator/touchpad-indicator
Testataan ajaa tila koneelle (huom touchpad-indicator on jo asennettuna, testataan vain meneekö komennot läpi)
	sudo salt '*' state.apply
Tulos:	

```
----------
          ID: base
    Function: pkgrepo.managed
        Name: ppa:atareao/atareao
      Result: True
     Comment: Configured package repo 'ppa:atareao/atareao'
     Started: 18:42:34.877492
    Duration: 998.124 ms
     Changes:   
----------
          ID: touchpad-indicator
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 18:42:35.875742
    Duration: 3.485 ms
     Changes:   
----------
          ID: /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf
    Function: file.managed
      Result: True
     Comment: File /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf is in the correct state
     Started: 18:42:35.879332
    Duration: 41.658 ms
     Changes:   
```

Testataan asentamalla toiselle koneelle (sama username)
Ensin lisätään top.sls tiedostoon toiselle koneelle myös touchpad-indicator -tila.
	sudoedit /srv/salt/top.sls
Joka näyttää lopulta tältä:

```
base:
  'asusG751jy':
    - touchpad-indicator/touchpad-indicator
  'hp1020g1':
    - touchpad-indicator/touchpad-indicator
```

Laitetaan tila käyttöön myös "uudelle" koneelle:
	sudo salt '*' state.apply
Tulos on virheellinen, koska hakemisto .conf tiedostolle puuttuu:

```
----------
          ID: base
    Function: pkgrepo.managed
        Name: ppa:atareao/atareao
      Result: True
     Comment: Configured package repo 'ppa:atareao/atareao'
     Started: 18:56:50.872931
    Duration: 1057.613 ms
     Changes:   
----------
          ID: touchpad-indicator.packages
    Function: pkg.installed
      Result: True
     Comment: The following packages were installed/updated: touchpad-indicator
     Started: 18:56:51.930635
    Duration: 7772.428 ms
     Changes:   
              ----------
              gir1.2-gconf-2.0:
                  ----------
                  new:
                      3.2.6-4ubuntu1
                  old:
              gir1.2-rsvg-2.0:
                  ----------
                  new:
                      2.40.20-2
                  old:
              python3-evdev:
                  ----------
                  new:
                      0.7.0+dfsg-2
                  old:
              python3-pyudev:
                  ----------
                  new:
                      0.21.0-1
                  old:
              python3-xlib:
                  ----------
                  new:
                      0.20-3
                  old:
              touchpad-indicator:
                  ----------
                  new:
                      2.2.1-0extras19.04.0
                  old:
----------
          ID: /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf
    Function: file.managed
      Result: False
     Comment: Parent directory not present
     Started: 18:56:59.706133
    Duration: 17.256 ms
     Changes:   
```

Muokataan touchpad-indicator.sls tiedostoa lisäämällä makedirs -optio:
	sudoedit /srv/salt/touchpad-indicator/touchpad-indicator.sls 
Tiedosto näyttää nyt tältä:

```
base:
  pkgrepo.managed:
    - name: ppa:atareao/atareao
    - dist: precise
    - file: /etc/apt/sources.list.d/atareao.list

touchpad-indicator.packages:
  pkg.installed:
    - fromrepo: stable
    - pkgs:
      - touchpad-indicator

/home/nikke/.config/touchpad-indicator/touchpad-indicator.conf:
  file.managed:
    - source: salt://touchpad-indicator/touchpad-indicator.conf
    - user: nikke
    - group: nikke
    - mode: 664
    - dir_mode: 775
    - makedirs: True
```

Laitetaan muokattu tila uudestaan päälle:
	sudo salt '*' state.apply
Tuloksesta huomaa, että kaikki kolme statea onnistui:

```
----------
          ID: base
    Function: pkgrepo.managed
        Name: ppa:atareao/atareao
      Result: True
     Comment: Configured package repo 'ppa:atareao/atareao'
     Started: 19:05:14.224850
    Duration: 933.147 ms
     Changes:   
----------
          ID: touchpad-indicator.packages
    Function: pkg.installed
      Result: True
     Comment: All specified packages are already installed
     Started: 19:05:15.158089
    Duration: 2.396 ms
     Changes:   
----------
          ID: /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf
    Function: file.managed
      Result: True
     Comment: File /home/nikke/.config/touchpad-indicator/touchpad-indicator.conf updated
     Started: 19:05:15.160551
    Duration: 15.065 ms
     Changes:   
              ----------
              diff:
                  New file
              group:
                  nikke
              mode:
                  0664
              user:
                  nikke
```

Tarkistetaan uudelta koneelta, että oikeudet sekä kansiossa, että kansion sisällä olevassa .conf tiedostossa ovat samat, kun käsin asennetussa:
	cd /home/nikke/.config
	ls -la|grep touch
Tulos:
	drwxrwxr-x  2 nikke nikke 4096 huhti 21 17:13 touchpad-indicator
	
	cd /home/nikke/.config/touchpad-indicator/
	ls -la|grep touch
Tulos:
	-rw-rw-r--  1 nikke nikke  664 huhti 21 18:49 touchpad-indicator.conf

Molemmilla koneilla näyttää oikeudet samoilta (eli oikeilta).
Testataan lopuksi vielä "uudella" koneella touchpad-indicatorin ja sen asetuksien toimivuus liittämällä hiiri tietokoneeseen.
Toiminnot näyttävät toimivan odotetusti.

##MarkDown. Tee tämän tehtävän raportti MarkDownina.

Tein tämän raportin Markdownina, käytin apuna [tätä ohjetta.](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet)

##Näytä omalla git-varastollasi esimerkit komennoista ‘git log’, ‘git diff’ ja ‘git blame’. Selitä tulokset.


##Tee tyhmä muutos gittiin, älä tee commit:tia. Tuhoa huonot muutokset.

