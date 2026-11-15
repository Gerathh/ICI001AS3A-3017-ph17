# h4 Pizza Fantasia

## Ympäristö

- 13.11.2026 tekijä **Jere Pellinen**
- **Kurssi:** Palvelinten hallinta ICI001AS3A-3017-ph17
- **Oppilaitos:** Haaga-Helia ammattikorkeakoulu
- **Opettaja:** Tero Karvinen
- **Ympäristö:** Debian 13, Hyper-V, paikallinen virtuaalikone

## x) Lue ja tiivistä


## x) Lue ja tiivistä

### 4.12.1 Size and Complexity of Some DSLs, s. 112

- DSL:t (Domain-Specific Language) voivat olla erittäin monimutkaisia ja isoja.
- Karvinen analysoi Saltin DSL:ää ja keräsi siitä dataa versiolla 2017.7.4 ”Nitrogen”.
- Tutkitussa Saltin versiossa oli 510 tilafunktiota. Niiden dokumentaatiossa oli yli 20 000 riviä ja yli 75 000 sanaa.
- Tutkimuksen mukaan Puppetissa oli 113 funktiota suoraan asennuksen jälkeen.
- Puppet käyttää omia käsitteitään, kuten virtuaalisia resursseja.

**Ajatuksia:** Pohdin tässä vaiheessa, mitä Karvinen tarkoittaa virtuaalisten resurssien käsitteellä ja miten se toimii käytännössä.

### 4.12.2 Use of DSL Functions in Case Configuration, s. 112–115

- Aineistona olivat Puppet-kielellä kirjoitetut tuotantokonfiguraatiot: United States Government Configuration Baseline (NIST, 2016) ja Mozilla Release Engineering Puppet Manifests (Mitchell et al., 2020).
- Mozillassa 12 toimintoa ja rakennetta kattoivat 87 % lasketuista esiintymistä.
- Tutkimuksessa tarkasteltiin koodia, ei komentojen suorituskertoja.
- Vaikka konfiguraationhallinnan (CM, Configuration Management) työkalut tarjoavat laajan kirjon toimintoja, tutkituissa aineistoissa vain muutamaa käytettiin laajasti.

**Ajatuksia:** Mielenkiintoista. Onko vain niin monta eri tapaa tehdä samat asiat?

### 4.12.3.1 Dependencies Between Main Functions, s. 115–117

- Kappaleessa käsitellään eri funktioiden riippuvuussuhteita. Tässä tapauksessa funktioilla tarkoitetaan resursseja.
- Confterossa, Tero Karvisen omassa konfiguraationhallintajärjestelmässä, tavoitteena on, ettei käyttäjän tarvitse ymmärtää syvällisesti funktioiden riippuvuuksien hallintaa.
- Järjestelmän tulisi olla idempotentti: muutoksia tehdään vain, jos järjestelmä ei ole jo oikeassa tilassa.
- `exec` ja tiedostojen hallinta (`file`, `directory`, `symlink`) ovat pohjimmaiset rakennuspalikat. Niiden varaan voidaan rakentaa esimerkiksi pakettien, palveluiden ja käyttäjien hallinta sekä edelleen korkeamman tason toimintoja.

**Ajatuksia:** Tulee mieleen makrot – ihmistä helpottamaan ja nopeuttamaan rutiininomaisia töitä.


## A) Räpylä

- Valitsin asennettavaksi Ecplipse Mosquitton. Se on kevyt, avoimen lähdekoddin viestinvälittäjä (message broker). Mosquitoo käyttää MQTTT protokollaa, ja mahdollistaa IoT laitteiden ja muitten laitteiden välisen kommunikoinnin.
- Aloitin asentamisen: 
- ![Asennus](images\asennusensin.png)

- Tässä vaiheessa kannattaa testata ovatko portit 1883, 1884 ja 1885 vapaana: 'sudo ss -ltnp '( sport = :1883 or sport = :1884 or sport = :1885 )'

- Muokkasin tämän jälkeen configurointitiedostoa 'sudo nano /etc/mosquitto/conf.d/h4.conf' ja lisäsin sinne kaksi riviä: 'listener 1884 127.0.0.1' 

ja 

'allow_anonymous true'



- Koitin käynnistää palvelua uudestaan, mutta tuli virheilmoitus.
- Aloitin vian selvittämisen 'systemctl status mosquitto' -komennolla joka näyttää ohjelman tilan. 
- Tämän jälkeen varmistin luomani configuraatiotiedoston oikeellisuuden 'sudo cat -n /etc/mosquitto/conf.d/h4.conf' 
- Sitten lokien tulostamisella 'sudo tail -n 40 /var/log/mosquitto/mosquitto.log'

![Virheilmoitus](images\error.png)
*Virheilmoitus sekä vianselvitystä*

- Suoritettuani 'sudo journalctl -u mosquitto -n 20 --no-pager' huomasin, että minulla oli kirjoitusvirhe h4.conf:ssa. 
- 'sudo nano /etc/mosquitto/conf.d/h4.conf' ja korjasin sielät toisen rivin kirjoitusvirheen 'allow anonymous true' --> 'allow_anonymous true'

-Tämän jälkeen mosquitto käynnistyi uudelleen ilman ongelmia

![Käynnistyi uudelleen](kaynnistyi.png)
*mosquitto käynnistyi uudelleen*

- Seuravaaksi testasin toimiiko MQTT viestit terminaalin kautta. Avasin kaksi terminaalia rinnakkain. Vasemmassa on vastaanottaja ja oikeassa lähetys.
- Vastaanottajan ikkunaan: 'mosquitto_sub -h 127.0.0.1 -p 1884 -t 'h4/pizza' -v -C 1'
- Lähettäjän ikkunaan: 'mosquitto_pub -h 127.0.0.1 -p 1884 -t 'h4/pizza' -m 'Jeren fantasiapizza: viestit toimivat portissa 1884'

![Viestit toimivat](images\viestitok.png)
*Viestit toimivat*


## b) Automaatti

### Uudelleenasennus

- Poistin mosquiton, jotaa voin asentaa sen Ansiblella. `sudo apt purge mosquitto`
- Asennusohjelma hupomautti, että luomaani konfiguraatiotiedostoa ei poistettu joten suoritin `sudo rm -f /etc/mosquitto/conf.d/h4.conf


![Mosquitton poisto](images/purge.png.png)
*Mosquitton poisto*

- Loin tehtävälle oman hakemiston ja Mosquitto toolin kansiot `mkdir -p ~/ansible-h4/roles/mosquitto/{tasks,handlers,templates,defaults}`
- Kirjoitin ~/ansible/h4.yml -tiedostoon rivit:
  
- `name: Manage local Mosquitto
  hosts: localhost
  connection: local
  become: true

  roles:
    - mosquitto`

- Testasin paikallisen yhteyden `ansible -i hosts.ini localhost -c local -m ansible.builtin.ping`
  
  ![Paikallinen yhteys toimii](images\pikallinen.png)
  *Paikallinen yhteys toimii*


- Tämän jälkeen loin oletusmuuttujan mmosquitton main.yml:_iin: mosquitto_port: 1884
- Sitten asetuspohja `nano ~/ansible/roles/mosquitto/templates/h4.conf.j2` teksti
 `listener {{ mosquitto_port }} 127.0.0.1
allow_anonymous true`


---
- name: Install Mosquitto and the client tools
  ansible.builtin.apt:
    name:
      - mosquitto
      - mosquitto-clients
    state: present

- name: Ensure configuration directory exists
  ansible.builtin.file:
    path: /etc/mosquitto/conf.d
    state: directory
    owner: root
    group: root
    mode: '0755'

- name: Config local MQTT listener
  ansible.builtin.template:
    src: h4.conf.j2
    dest: /etc/mosquitto/conf.d/h4.conf
    owner: root
    group: root
    mode: '0644'
  notify: Restart Mosquitto

- name: Ensure Mosquitto is running and enabled
  ansible.builtin.service:
    name: mosquitto
    state: started
    enabled: true


-Muokkasin vielä `nano ~/ansible/roles/mosquitto/handlers/main.yml` ja kirjoitin sinne:

`- name: Restart Mosquitto
  ansible.builtin.service:
    name: mosquitto
    state: restarted`

- Tarkistin tämän jälkeen syntaksin: `ansible-playbook -i hosts.ini h4.yml --syntax-check` ja kaikki oli ok. 
- Sitten ajoin`ansible-playbook -i hosts.ini h4.yml --ask-become-pass`

![Syntaksi ok](images\syntaksiok.png)
*Syntaksi oli ok*

- Kirjoitin jälleen vasempaan terminaaliin `mosquitto_sub -h 127.0.0.1 -p 1884 -t 'h4/pizza' -v -C 1`
- Oikeaan terminaallin `mosquitto_pub -h 127.0.0.1 -p 1884 -t 'h4/pizza' -m 'Ansible-asennuksen viestitesti'` 

![Asennus ok ansiblen kautta](images\ansiblemosquittook.png)
*Asennus Ansiblen kautta toimii*

## c) Asetus

- Muutin tiedostossa nano `~/ansible/roles/mosquitto/defaults/main.yml` arvoksi `mosquitto_port: 1885`
- Ajoin uudelleen `ansible-playbook -i hosts.ini h4.yml --ask-become-pass`
- Sitten ajoin `cat /etc/mosquitto/conf.d/h4.conf` ja `sudo ss -ltnp '( sport = :1883 or sport = :1884 or sport = :1885 )'`
- Muutokset näkyivät oikein.
- Tarkistin muutosten toimivuuden vielä käytännössä: vasemmassa temrinaalissa `mosquitto_sub -h 127.0.0.1 -p 1885 -t 'h4/pizza' -v -C 1`
- Oikeassa terminaalissa: `mosquitto_pub -h 127.0.0.1 -p 1885 -t 'h4/pizza' -m 'Portin vaihto onnistui: 1885'`

![Portin vaihto onnistui](images\portinvaihtook.png)
*Portin vaihto onnistui*


### Lähde

- [Karvinen 2023: Configuration Management of Distributed Systems over Unreliable and Hostile Networks](https://terokarvinen.com/palvelinten-hallinta/Karvinen-2023-Configuration-Management-of-Distributed-Systems-over-Unreliable-and-Hostile-Networks.pdf), kohdat 4.12.1, 4.12.2 ja 4.12.3.1, s. 112–117.
- [mosquitto kuvaus] (https://projects.eclipse.org/projects/iot.mosquitto)
- [mosquitto Documentation] (https://mosquitto.org/documentation/)

###











### Lähteet 

- [Karvinen 2026: Palvelinten hallinta - h3 Pizza Fantasia](https://terokarvinen.com/palvelinten-hallinta/) - Tehtävänanto ja ohjeet.
- [](https://t) - 
