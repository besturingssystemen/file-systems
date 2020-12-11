# File systems <!-- omit in toc -->

## Inhoudstafel <!-- omit in toc -->

- [Voorbereiding](#voorbereiding)
- [GitHub classroom](#github-classroom)
- [Introductie](#introductie)
- [Block device](#block-device)
- [Disk layout](#disk-layout)
  - [Superblock](#superblock)
- [Performantie](#performantie)
  - [Buffer cache](#buffer-cache)
  - [Transaction log](#transaction-log)
- [Disk space management](#disk-space-management)
  - [Bitmap](#bitmap)
- [Files](#files)
  - [Inodes](#inodes)
  
## Voorbereiding

Ter voorbereiding van deze zitting worden jullie verwacht:

* De oefenzitting Synchronizatie te hebben voltoold
* Hoofdstuk 8 van het [xv6 boek](https://github.com/besturingssystemen/xv6-riscv) te hebben gelezen.

## GitHub classroom

<!-- TODO -->

## Introductie

Besturingssystemen werken met een geheugenhiërarchie.
Op het moment dat een OS actief is, zit er bijvoorbeeld code en data in het RAM, in caches en registers.
Al dit geheugen is echter vluchtig.
Wanneer je de stroom van de machine uittrekt, is al deze data plots weg.

Code en data die we willen bewaren voor een lange tijd, ook na het heropstarten van een machine, bewaren we op een opslagmedium.
Op dit opslagmedium staat de code van het Besturingssysteem, alle data, de bestanden die een gebruiker op zijn machine heeft staan, enzovoort.

Er zijn vele soorten opslagmediums, elk met hun eigen capaciteit, snelheid en eventueel andere voor- en nadelen.
Al deze mediums kunnen gekoppeld worden aan een computer.
Zo heb je [Hard disk drives](https://en.wikipedia.org/wiki/Hard_disk_drive), [Solid-state drives](https://en.wikipedia.org/wiki/Solid-state_drive), [USB flash drives](https://en.wikipedia.org/wiki/USB_flash_drive), [Floppy disk drives](https://en.wikipedia.org/wiki/Floppy_disk), enzovoort.
Dit soort apparaten worden ook welk [*mass storage devices*](https://en.wikipedia.org/wiki/Mass_storage) genoemd.

> :information_source: Bij het opstart van een PC zal typisch de [BIOS](https://en.wikipedia.org/wiki/BIOS) de opstartcode (boot loader) lezen uit een drive en in het RAM-geheugen plaatsen.
> Vanuit RAM kan de processor deze code uitvoeren.
> Deze boot loader zal het RAM geheugen van de machine initialiseren via een [bootstrap-proces](https://en.wikipedia.org/wiki/Bootstrapping).
> Gedurende dit proces kunnen er eventueel ook nog andere nodige delen van het OS uit de drive gelezen worden.

Elk van deze mediums maken het mogelijk grote hoeveelheden data te bewaren.
Deze data moet georganiseerd worden.
Er moet een manier zijn om te weten welke data bij welk bestand hoort.
Deze organisatie gebeurt met behulp van *file systems*.
File systems kunnen op [vele verschillende manieren](https://en.wikipedia.org/wiki/Comparison_of_file_systems) geïmplementeerd worden.
In deze zitting zullen we dieper ingaan op het custom file system van xv6.

## Block device

Elk van deze opslagmedia kunnen we opdelen in blokken vast vaste grootte.
De blokken kan je vervolgens elk een adres voorstellen.
De verschillende apparaten kunnen we dus abstract voorstellen als apparaten waarnaar je blokken geheugen kan lezen en schrijven.
Deze abstracte apparaten noemen we block devices.

xv6 neemt aan dat het file system bewaard wordt op zo'n block device.
De grootte van een blok is [in xv6 ingesteld op `BSIZE` bytes](block-size).
Het block device waarop xv6 zijn file system bewaart, en dat dus opgedeeld is in blokken van grootte `BSIZE`, wordt de *disk* genoemd.

> :information_source: Merk op dat deze *disk* een hard-drive, solid state drive, USB drive, ... zou kunnen zijn. In qemu emulator wordt een hard drive geëmuleerd op basis van een *image* file (`fs.img`). Deze image bevat de inhoud van de virtuele disk.

## Disk layout

We weten dus dat onze disk opgedeeld is in blokken van grootte `BSIZE`.
Welke data kan je terugvinden in welke blokken wordt vastgelegd in de disk layout.
Onderstaande afbeelding toont deze layout.

![xv6-disk-layout](img/disk-layout-xv6.png)

De allereerste blok van de disk, blok 0, wordt gebruikt om de *boot code* te bewaren.
We noemen dit de [boot sector](https://en.wikipedia.org/wiki/Boot_sector).
Het is conventie om de boot sector te bewaren in de allereerste blok van een mass storage device.
Deze conventie laat ons toe om andere boot loaders te installeren of om de boot loader te bewerken, om zo verschillende soorten besturingssystemen te op te kunnen starten op éénzelfde machine, vanuit verschillende soorten mass storage devices.
De boot sector is nog geen deel van het file system zelf.

> :information_source: Mass storage devices kunnen opgedeeld worden in verschillende partities, elk met hun eigen bootgedeelte. Op sector 0 vinden we in dat geval de [Master Boot Record](https://en.wikipedia.org/wiki/Master_boot_record), met informatie over de verschillende partities op die specifieke schijf. Indien een partitie een OS bevat zal deze partitie vervolgens een [Volume Boot Record](https://en.wikipedia.org/wiki/Volume_boot_record) hebben met daarin de boot code van het OS op die partitie. Wanneer de xv6 boek of deze oefenzitting spreekt over een disk, moet je dit voorstellen als een partitie, niet als de volledige harde schijf. De boot sector van xv6 zou dus in de VBR zitten van een gepartitioneerde schijf.

### Superblock

Blok 1 is de eerste blok van het filesystem zelf.
In xv6 wordt deze blok de superblock genoemd.
De superblock bevat meta-informatie over het filesystem.
Op basis van de superblock kan je de volledige disk layout van het file system achterhalen.

* Bekijk [`struct superblock`](superblock) in `kernel/fs.h`. Vergelijk met bovenstaande figuur waarin de lay-out van de disk wordt beschreven.

Het veld `magic` bevat een [magic number](https://en.wikipedia.org/wiki/Magic_number_(programming)), dat gebruikt wordt om het xv6 file system te identificeren. De magic number van het xv6 file system bestaat uit de bytes `0x40 0x30 0x20 0x10`, ofwel het getal `0x10203040` voorgesteld met de [little-endian](https://en.wikipedia.org/wiki/Endianness) byte order.
Magic numbers in bestanden en file systems worden gebruikt om de verschillende soorten van elkaar te onderscheiden. Zo start bevoorbeeld elk [ELF-bestand](https://en.wikipedia.org/wiki/Executable_and_Linkable_Format) met de bytes `0x7F 0x45 0x4c 0x46`, de magic number van ELF-files.

> **:question: Voer `make` uit in de xv6-repo. Voer vervolgens het commando `hd fs.img | head -4` uit. Het commando hex dump (`hd`) toont de inhoud van een bestand byte per byte. Kan je de magic number van xv6 terugvinden? Wat is het byte-adres van de eerste byte van deze magic number? Waarvan komt deze waarde?**

Het tweede veld `size` bevat de grootte, uitgedrukt in aantal blokken, van de volledige file system image.

> **:question: Voer nogmaals `hd fs.img | head -4` uit. Bereken de grootte in bytes van het file system op basis van het tweede veld `size`. Voer vervolgens `ls -l` uit en vergelijk met de grootte van `fs.img`.**
>
> :bulb: Het type `uint` is telkens 4 bytes groot, de waarden staan opgeslagen in [little-endian byte order](https://en.wikipedia.org/wiki/Endianness).

Vervolgens worden `nblocks`, `ninodes` en `nlog` bewaard, die respectievelijk het aantal data blokken, inodes en log blokken weergeven.
Ten slotte bevatten `logstart`, `inodestart` en `bmapstart` de blok-adressen waar de eerste blok van respectievelijk de `log`, de `inodes` en de `bitmap` secties.
We leggen in de komende secties uit wat de `log`, `inodes` en `bitmap` secties net voorstellen.

## Performantie

Lezen en schrijven naar mass storage kan erg traag zijn.
Indien je bijvoorbeeld gebruik maakt van een harde schijf, zal deze schijf eerst fysiek moeten roteren en eventueel van track verwisselen om de juiste sector te selecteren.
Dat kan allemaal lang duren.
Hoewel een Solid State Drive al een stuk sneller is, is een leesoperatie uit een SSD schijf nog steeds enkele grootte-ordes trager dan een leesoperatie uit RAM geheugen.

> :information_source: De tijd tussen een leesoperatie en het moment dat de data beschikbaar is vanuit RAM geheugen [wordt gemeten in nanoseconden](https://en.wikipedia.org/wiki/CAS_latency).
Voor een SSD [wordt dit gemeten in microseconden](https://blocksandfiles.com/2020/09/08/seven-attempts-to-speed-processing-with-faster-storage/) (1 microseconde = 1000 nanoseconden). Voor een HDD [gaat dit zelfs over miliseconden](https://en.wikipedia.org/wiki/Hard_disk_drive_performance_characteristics) (1 miliseconde = 1000 microseconden).

### Buffer cache

Om ervoor te zorgen dat lees- en schrijfoperaties naar bestanden veel sneller kunnen verlopen, maakt het file system van xv6 gebruik van een [*cache*](https://en.wikipedia.org/wiki/Cache_(computing)).
Wanneer een geheugenblok gelezen wordt, wordt deze geheugenblok in het RAM-geheugen geplaatst, in de *buffer cache*.
Vanaf dan kunnen alle lees- en schrijfoperaties rechtstreeks via het RAM-geheugen verlopen, in plaats van via de harde schijf.

De buffer cache wordt voorgesteld door een *doubly linked list* van buffers (gecachte blokken), gesorteerd zodat de *least recently used* buffer achteraan (`head->prev`) in de lijst staat en de *most recently used* buffer dus vooraan (`head->next`) staat.

* Bekijk de functie [`bget`][bget] in `kernel/bio.c`. Deze functie wordt gebruikt door [`bread`][bread] en [`bwrite`][bwrite] om een buffer overeenkomstig met een gegeven bloknummer op te vragen. Indien de blok niet gecacht is wordt er gezocht naar een ongebruikte buffer in de buffer cache en wordt deze buffer toegewezen aan een specifieke blok.
* Bekijk de functie [`bread`][bread] en [`bwrite`][bwrite]. `bread` vraagt aan `bget` een buffer voor een specifiek bloknummer. Indien deze buffer zonet door `bget` hergebruikt werd bevatte deze nog foute data van een andere blok (`b->valid == 0`), en wordt de juiste blokdata van de disk gelezen. `bwrite` zorgt ervoor dat de data in een buffer naar de correcte blok in de mass storage wordt geschreven.

Het gevolg van het invoeren van een cache laag is dat lees- en schrijfoperaties niet meer rechtstreeks naar de harde schijf verlopen.
Pas wanneer buffers expliciet weggeschreven worden met `bwrite` is een aanpassing van de inhoud van een blok effectief bewaard in de long term storage.

Dit zorgt voor een grote uitdaging.
Een besturingssysteem kan op ieder moment crashen, ter gevolg van een fout in de code of gewoon door stroom die wegvalt.
Op dat moment kunnen er echter nog een hoop aangepaste disk blokken in de cache zitten met nieuwe data, die nog niet correct weggeschreven zijn naar de onderliggende mass storage.
Indien bepaalde operaties bestanden bewerken en sommige schrijfoperaties zijn reeds op de harde schijf doorgevoerd en andere operaties niet, kan het file system in een inconsistente toestand terecht komen.

### Transaction log

xv6 lost dit op door middel van [transacties](https://en.wikipedia.org/wiki/Transaction_processing).
Een transactie groepeert verschillende schrijfoperaties.
Er wordt gegarandeerd dat ofwel alle schrijfoperaties in een transactie uitgevoerd worden, ofwel geen enkele.
Dit wordt gegarandeerd via een *transaction log*.

Alle gegroepeerde schrijfoperaties in een transactie worden eerst geschreven naar een aparte regio van de disk, de log.
Wanneer al deze wijzigingen zich op de disk bevinden (maar dus nog niet in de correcte block), weten we zeker dat de nieuwe data effectief weggeschreven kan worden.
Een crash kan er namelijk niet meer voor zorgen dat data uit de transactie verloren gaat, alle data staat op non-volatile storage.
Op dat moment kan de transactie gecommit worden, en kunnen alle blokken van de log geschreven worden naar de correcte blok op de disk.

Indien er zich een crash voordoet tijdens de commitfase is dit geen probleem.
Bij het heropstarten van de machine zal gezien worden dat de transactie log een transactie bevat die nog niet gecommit was, en zal deze transactie opnieuw ingezet worden.
Er is geen data verloren gegaan.
Transacties garanderen dus dat alle schrijfoperaties in één transactie ofwel samen uitgevoerd worden, ofwel niet uitgevoerd worden.

Stel dat de machine crashte en de transactie was nog niet volledig klaar (slechts enkele writes stonden in de log, andere writes dan weer niet) wordt de transactie dus gewoon nooit uitgevoerd, want op dat moment kan het zijn dat het comitten van deze incomplete transactie ervoor zou zorgen dat het file system in een inconsistente state terecht komt.

<!-- TODO codevoorbeelden toevoegen -->
## Disk space management

### Bitmap

## Files

### Inodes

<!-- TODO -->

<!-- TODO -->

[block_size]:https://github.com/besturingssystemen/xv6-riscv/blob/02ca399d0590a57d9ba05fcf556546141a5e2a09/kernel/fs.h#11
[superblock]:https://github.com/besturingssystemen/xv6-riscv/blob/02ca399d0590a57d9ba05fcf556546141a5e2a09/kernel/fs.h#19
[bget]:https://github.com/besturingssystemen/xv6-riscv/blob/bss/kernel/bio.c#55
[bread]:https://github.com/besturingssystemen/xv6-riscv/blob/bss/kernel/bio.c#91
[bwrite]:https://github.com/besturingssystemen/xv6-riscv/blob/bss/kernel/bio.c#105