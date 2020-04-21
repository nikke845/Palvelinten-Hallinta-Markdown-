# Palvelinten hallinta viikko3

Tätä dokumenttia saa kopioida ja muokata GNU General Public License (versio 2 tai uudempi) mukaisesti. [http://www.gnu.org/licenses/gpl-3.0.html](http://www.gnu.org/licenses/gpl-3.0.html)
Pohjana Tero Karvinen 2020: [Palvelinten Hallinta – Spring 2020](http://terokarvinen.com/2020/configuration-managment-systems-palvelinten-hallinta-ict4tn022-spring-2020/)

Tehtävän teko aloitettu 21.4.2020 noin klo 17.00.

## Uusi Salt moduuli

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

Varmistetaan, että tiedosto löytyy:

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

```
base:
  'hp1020g1':
    - touchpad-indicator/touchpad-indicator
```

Testataan ajaa tila koneelle (huom. touchpad-indicator on jo asennettuna, testataan vain meneekö komennot läpi):

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

Testataan asentamalla toiselle koneelle (sama username).
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

## MarkDown. Tee tämän tehtävän raportti MarkDownina.

Tein tämän raportin Markdownina, käytin apuna [tätä ohjetta.](https://github.com/adam-p/markdown-here/wiki/Markdown-Cheatsheet) Raportti löytyy myös [Githubista.](https://github.com/nikke845/Palvelinten-Hallinta-Markdown-/blob/master/viikko3.md)

## Näytä omalla git-varastollasi esimerkit komennoista ‘git log’, ‘git diff’ ja ‘git blame’. Selitä tulokset.

### git log

Komennolla pystyy katsomaan commit historiaa. Komennolla on paljon eri optioita joita voi katsoa vaikka [täältä.](https://git-scm.com/docs/git-log)

	git log

Tulos:

```
commit aade1feac999041e55ec80a5d55b9924ad0e35fa (HEAD -> master, origin/master, origin/HEAD)
Author: nikke845 <nvuorivirta@gmail.com>
Date:   Tue Apr 21 20:16:20 2020 +0300

    e) Tee tyhmä muutos

commit 0d97ebc6b79adaeef3d120f9e6aaf175fcd13e31
Author: nikke845 <nvuorivirta@gmail.com>
Date:   Tue Apr 21 20:07:06 2020 +0300

    Markdown changes

commit d5a1657a1f4d3b8d31005d89feb8ea140133d8b8
Author: nikke845 <nvuorivirta@gmail.com>
Date:   Tue Apr 21 20:00:12 2020 +0300

    uusi moduuli valmis

commit 8eb2ea241354e08e8ccf77af2a4d5a2c3fbe946a
Author: nikke845 <42818108+nikke845@users.noreply.github.com>
Date:   Tue Apr 21 19:56:45 2020 +0300

    Initial commit
```

Komennon analysointi:

```
commit aade1feac999041e55ec80a5d55b9924ad0e35fa (HEAD -> master, origin/master, origin$
Author: nikke845 <nvuorivirta@gmail.com>
Date:   Tue Apr 21 20:16:20 2020 +0300

    e) Tee tyhmä muutos
```

```
commit id
author and email
timestamp

commit message
```

### git diff

Komennolla voi vertailla mm. committeja ja työhakemistoa. [Täältä](https://git-scm.com/docs/git-diff) löytyy lisää optioita.
Etsitään lyhennetyt commit id:t :

	git log --oneline

Tulos:

```
aade1fe (HEAD -> master, origin/master, origin/HEAD) e) Tee tyhmä muutos
0d97ebc Markdown changes
d5a1657 uusi moduuli valmis
8eb2ea2 Initial commit
```

Verrataan committia ´aade1fe´ committiin ´0d97ebc´:

	git diff aade1fe 0d97ebc

Tulos:

```
diff --git a/viikko3.md b/viikko3.md
index 9791bb5..ae8778a 100644
--- a/viikko3.md
+++ b/viikko3.md
@@ -1,11 +1,11 @@
-# Palvelinten hallinta viikko3
+#Palvelinten hallinta viikko3
 
 Tätä dokumenttia saa kopioida ja muokata GNU General Public License (versio 2 tai uudempi) mukaisesti. [http://www.gnu.org/licenses/gpl-3.0.html](http://www.gnu.org/licenses/gpl-3.0.html)
 Pohjana Tero Karvinen 2020: [Palvelinten Hallinta – Spring 2020](http://terokarvinen.com/2020/configuration-managment-systems-palvelinten-hallinta-ict4tn022-spring-2020/)
 
 Tehtävän teko aloitettu 21.4.2020 noin klo 17.00.
 
-## Uusi Salt moduuli
+##Uusi Salt moduuli
 
 Moduuli on asennettu [tämän ohjeen mukaan.](https://itsfoss.com/disable-touchpad-when-mouse-used/)
 
@@ -298,59 +298,13 @@ Molemmilla koneilla näyttää oikeudet samoilta (eli oikeilta).
 Testataan lopuksi vielä "uudella" koneella touchpad-indicatorin ja sen asetuksien toimivuus liittämällä hiiri tietokoneeseen.
 Toiminnot näyttävät toimivan odotetusti.
 
-## MarkDown. Tee tämän tehtävän raportti MarkDownina.
+##MarkDown. Tee tämän tehtävän raportti MarkDownina.
...
```
### git blame

Komento näyttää tiedoston commit id:n ja commitin tekijän, sekä aikaleiman rivi riviltä. Komento toimii vain yksittäisille tiedostoille. Lisää optioita löydät [täältä.](https://git-scm.com/docs/git-blame)

Demo:

	git blame viikko3.md

Tulos:

```
aade1fea (nikke845          2020-04-21 20:16:20 +0300   1) # Palvelinten hallinta viikko3
d5a1657a (nikke845          2020-04-21 20:00:12 +0300   2) 
d5a1657a (nikke845          2020-04-21 20:00:12 +0300   3) Tätä dokumenttia saa kopioida ja muokata GNU General Public License (versio 2 tai uudempi) mukaisesti. [http://www.gnu.org/licenses/gpl-3.0.html](http://www.gnu.org/licenses/gpl-3.0.html)
d5a1657a (nikke845          2020-04-21 20:00:12 +0300   4) Pohjana Tero Karvinen 2020: [Palvelinten Hallinta – Spring 2020](http://terokarvinen.com/2020/configuration-managment-systems-palvelinten-hallinta-ict4tn022-spring-2020/)
d5a1657a (nikke845          2020-04-21 20:00:12 +0300   5) 
d5a1657a (nikke845          2020-04-21 20:00:12 +0300   6) Tehtävän teko aloitettu 21.4.2020 noin klo 17.00.
d5a1657a (nikke845          2020-04-21 20:00:12 +0300   7) 
aade1fea (nikke845          2020-04-21 20:16:20 +0300   8) ## Uusi Salt moduuli
```

Komennon tuloksen analysointia:

```
aade1fea (nikke845          2020-04-21 20:16:20 +0300   1) # Palvelinte...

commit id, commit author, timestamp, row number, row content
```
## Tee tyhmä muutos gittiin, älä tee commit:tia. Tuhoa huonot muutokset.

Tehdään viikko3.md tiedostoon muutos:

	nano viikko3.md

Johon laitetaan seuraavaa:

	Tyhmä muutos

Katsotaan status:

	git status

Tulos:

```
On branch master
Your branch is up to date with 'origin/master'.

Changes not staged for commit:
  (use "git add <file>..." to update what will be committed)
  (use "git checkout -- <file>..." to discard changes in working directory)

	modified:   viikko3.md

no changes added to commit (use "git add" and/or "git commit -a")
```

Kumotaan muutokset (siirrytään viimeiseen committiin):

	git reset --hard

Tulos:

	HEAD is now at 0d97ebc Markdown changes

Katsotaan vielä status:

	git status

Tulos:

```
On branch master
Your branch is up to date with 'origin/master'.

nothing to commit, working tree clean
```

Tehtävän teko lopetettu noin klo 21.00. Tehtävän aikana pidin muutaman lyhyen tauon.
