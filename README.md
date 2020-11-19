# template-daemon
Runko *nix daemon-prosessille

# Käyttö
C-koodin kääntäminen:
```gcc -o firstdaemon daemonize.c```

Daemon prosessin käynnistäminen:
```./firstdaemon```

Tarkista että prosessi on ajossa taustalla:
```ps -xj | grep firstdaemon```

# Daemon prosessin toiminta
## Yleistä
Daemon prosessilla tarkoitetaan prosessia joka on ajossa käyttöjärjestelmän taustalla, sen sijaan että se olisi jatkuvasti interaktiivisessa vuorovaikutuksessa käyttäjän kanssa. Tyypillisesti prosessin nimen perään on lisätty kirjain 'd' jotta käyttäjä tietää että kyseessä on taustaprosessi, eli daemon. Esimerkkejä daemon prosesseista ovat esimerkiksi unix käyttöjärjestelmien os-tasoisesta lokituksesta huolehtiva syslogd, ja sitten sshd, joka taustalla odottaa saapuvia ssh-yhteyksiä. Vastaava ohjelma toimintaperiaatteeltaan Windows ympäristössä on palvelu (engl. Service).

## Toiminta
Daemon prosessi luodaan yleensä isäntäprosessista haarauttamalla, eli forkkaamalla. Isäntäprosessi toimii alustavana prosessina, eli 'init(ialize)-prosessina', jonka tehtävä on alustaa taustaprosessin toiminta. Kun isäntä on suorittanut forkkauksen itsestään ja luonut taustaprosessin, se suorittaa exit(<exit_code>) systeemikutsun, eli pyytää käyttöjärjestelmää tappamaan oman prosessinsa.

> R27: if(pid > 0)
> R28:  exit(EXIT_SUCCESS);

Samaisen systeemikutsun mukana prosessi kertoo käyttöjärjestelmälle ja käyttäjälle exit_code -parametrilla onnistuiko taustaprosessin luominen vai päätyikö se virheeseen. Tämä logiikka rakennetaan init-koodiin process id (pid) vertailemalla yllä olevaan tapaan. Jos PID on suurempi kuin 0, eli init-prossein ID, tarkoittaa se sitä että käyttöjärjestelmä on myöntänyt sille prosessitaulusta oikean ID:n ja sallinut sen suorituksen. Exit -komennon suorittaminen on tärkeää seuraavista syistä. Mikäli taustaprosessi on käynnistetty shell-komentona, kun prosessi lopettaa toimintansa, lopettaa shell odottamasta prosessilta vastausta ja täten sallii käyttäjän jatkavan muuta toimintaansa. Lisäksi taustaprosessi ei saa olla oman prosessiperheen isäntä (engl. parent/leader), jotta setsit() -funktiokutsulla voidaan luoda halutusti uusi sessio.

> R31: if (setsid() < 0)
> R32:    exit(EXIT_FAILURE);

Funktiokutsu setsid() täydentää prosessin forkkauksen. Kutsun avulla taustaprosessi kiinnitetää uuteen sessioon, täten luoden ko. taustaproessista oman prosessiperheensä isännän. Lisäksi uuden session alaisena taustaprosessi ei enää ole riippuvainen sen luoneesta komentotulkista, eikä komentotulkki toisaalta myöskään prosessin suorituksesta. Joissakin käyttöjärjestelmissä on suositeltua suorittaa uusi fork() funktion kutsu setsid() funktiokutsun jälkeen.

> R40: pid = fork();

System V perustuvissa käyttöjärjestelmissä halutaan uudella forkkauksella tuhota sessioperheen isäntä, jatkaa suoritusta tämän session lapsena. [TARKENNA]

Hyvän tavan mukaisesti taustaprosessille tulisi määrittää se, miten se luo uusia tiedostoja käyttöjärjestelmän levyjärjestelmään. Tai tarkemmin ottaen, millaisilla oikeuksilla. Kuten tiedostossa daemonize.c määritellään taustaprosessille:

> R51: umask(0);

jossa tiedosto-oikeudet määritellään funktiokutsun parametrina kolmen bitin avulla. Eli tässä tapauksessa tiedoston omistajalle luku, kirjoitus ja suoritus oikeudet (rwx), kun taas ryhmän ja muiden oikeudet jäävät määrittelemättä, jolloin ne ovat vakioarvot, eli 7. Jolloin umask(0) on ekvalentti umask(077) funktiokutsun kanssa.

Seuraavaksi alustetaan taustaprosessin työkansio, eli tiedostojärjestelmän polku jossa prosessi operoi.

> R55: chdir("/");

Tämän perusteella voidaan viitata absoluuttisen tiedostopolun lisäksi relatiivisella tiedostopolulla. Eli sen sijaan, että viitataan tiedostoihin niiden koko polulla (absoluuttinen), voidaan viitata esimerkiksi työkansiossa oleviin tiedostoihin relatiivisesti './my_daemons.log' -notaatiolla, jolla viitataan suoraan työkansiossa olevaan log-tiedostoon. Lisäksi siirtämällä työkansion johonkin määritettyyn sijaintiin, voidaan varmistua siitä, että taustaprosessi ei ole ajossa jonkin liitetyn levyjärjestelmän kansiopolussa. Jolloin levyjärjestelmän poistaminen ei onnistu ilman virheitä taustaprosessin ajossa, ja toisaalta taustaprosessin suoritus saattaa vaikettaa levyjärjestelmän poistamista.

Lopuksi lapsiprosessi siivoa mahdollisia perittyjä tiedosto viitteittä (engl. File Descriptors "FD") jotka voivat osoittaa mahdollisiin komentotulkkeihin tai muihin tiedostoihin. Tällä varmistetaan standard input, standard output ja standard error viitteiden oikeallinen toiminta, ja toisaalta vältetään mahdolliset kummallisuudet taustaprosessin toiminnassa.

> R58: int x;
> R59: for (x = sysconf(_SC_OPEN_MAX); x>=0; x--)
> R61:    close (x);

## Daemon prosessit käyttöjärjestelmän käynnistyksen yhteydessä

#  Viitteet
https://notes.shichao.io/apue/ch13/
