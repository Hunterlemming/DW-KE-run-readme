

User Manual
===========

Teljes kimenet előállítása
--------------------------

1. Contract backend futtatása
2. Tender backend futtatása (a Contractéhoz hasonlóan, adatbázis lekapcsolása nélkül)
3. Buyer\_ID egységesítés
4. Extended output előállítása
5. Zippeld össze a Contract, a Tender, az Extended és a Merged outputot.

Contract backend futtatása
--------------------------

**Alapok**
* Buildelj: `./build.sh`
* Indítsd el a contract adatbázist: `docker-compose up -d dw-database-c`
* Indítsd el a nyulat: `docker-compose up -d dw-rabbit`
* Hírdesd meg a portokat: `./debug_kenya_c.sh`
* Állítsd le a workereket: `./killworkers.sh`

**Tömd meg a nyulat raw\_data-val!**
* Lépj be a RabbitFeeder-eager mappába: `cd ~/Path/To/RabbitFeeder-eager`
* Írd át a SendToRabbit.mjs-t, hogy a Contract-nyúl által meghírdetett queue-ra pakoljon: `"routing_key":"dfid_1.0_development_eu.datlab.worker.ke.raw.KEContractDownloader"`
* Indítsd el a docker-környezetet: `./run-docker.sh`
* Futtasd a scriptet: `node index.mjs`
* Miután végzett, lépj ki a docker-környezetből: `exit`
* Lépj vissza a backend mappájába: `cd ~/Path/To/DW-Backend`
* Indítsd el újra a Contract workereket: `./debug_kenya_c.sh`

**Indikátorok frissítése master\_tender rekordokban**
* Az adatok sikeres (elő-)feldolgozása után állítsd le a workereket: `./killworkers.sh`
* Navigálj az IndicatorWorkerFeeder mappájába: `cd ~Path/To/IndicatorWorkerFeeder`
* Indítsd el a docker-környezetet: `./run-docker.sh`
* Futtasd a scriptet: `node index.mjs`
* Miután végzett, lépj ki a docker-környezetből: `exit`
* Lépj vissza a backend mappájába: `cd ~/Path/To/DW-Backend`
* Indítsd el újra a Contract workereket: `./debug_kenya_c.sh`
* Ha az előző lépéssorozat sikeresen lezajlott, állítsd le a workereket: `./killworkers.sh`
* Állítsd le a nyulat: `docker stop dw-rabbit-kenya`
* Töröld a nyúl belső adatait: `docker rm dw-rabbit-kenya`
* Készíts egy Contract-Output mappát: `mkdir ~/Path/To/Contract/Output/Directory`
* Mozgasd az elkészült extra adatokat tartalmazó csv-ket a Contract-Output mappába: `mv ./csv-output/*.csv ~/Path/To/Contract/Output/Directory/`

**Flatten-Tool**
* Váltsál a backendből a flatten-tool tender\_analytical mappájába: `cd flatten-tool/tender_analytical/`
* Indítsd el a Python környezetet: `source flatten_venv/bin/activate`
* Futtasd a gen\_all\_analytics scriptet, Contractnak megfelelően paraméterezve (ke1 - Contract, ke2 - Tender): `python gen_all_analytics.py ke1`
* Miután végzett, lépj ki a Python környezetből: `deactivate`
* Exportáld ki a flatten.csv-t a mellékelt script (_db\_script_) futtatásával (pl.: DBeaver)
* A Script végén látod, hogy hova mentette a csv-t dockeren belül. Másold át innen a Contract-Output mappába: `docker cp dw-database-ke-contract:/Path/To/flat.csv ~/Path/To/Contract/Output/Directory/`

**Exportáld ki az Exported-JSon-t az adatbázisból**
* Navigálj vissza a backend mappájába: `cd ~Path/To/DW-Backend`
* Indítsd el a json.sh scriptet: `./json.sh`
* Nyiss egy új terminált és lépj be a BulkJsonExporter mappájába: `cd Path/To/BulkJsonExporter`
* Indítsd el a docker-környezetet: `./run-docker.sh`
* Futtasd a scriptet: `node index.mjs`
* Miután végzett, módosítsd az elkészül output hozzáférési jogosultságait: `chown 1000:1000 exported-tender-data.json`
* Másold az exportált flatten-tool csv output-mappájába: `cp ./exported-tender-data.json ~/Path/To/Contract/Output/Directory`
* Lépj ki a docker-környezetből: `exit`
* Zárd be a BulkJsonExporter terminálját, majd állítsd le a json.sh-t a backenden: `[Ctrl + C]`
* Állítsd le a Contract adatbázist: `docker stop dw-database-ke-contract`

Buyer\_ID egységesítés
---------------------

* Navigálj a dw-Kenya-buyer-id-standardizer mappájába: `cd ~/Path/To/dw-Kenya-buyer-id-standardizer`
* Indítsd el a Python környezetet: `source kenya-buyeris-venv/bin/activate`
* Másold a Contract Exported.json-t és Flat.csv-t a data/input mappába: `cp ~/Path/To/Contract/Output/Directory/{Exported.json,Flat.csv} .Kenya-Buyer-ID-Standardizer/data/output/`
* Menj vissza a Kenya-Buyer-ID-Standardizer könyvtárába: `cd ../..`

A program 3 fő része a main függvényből indul:\
&nbsp;&nbsp;**export\_buyer\_id\_mapping** - Első futtatáskor elkészíti az input-contract-json és a működő tender-adatbázis buyer\_id-i közötti mappelési táblát.\
&nbsp;&nbsp;**json\_data.renew_buyer_ids** - A mappelés alapján kicseréli az input-json-ben (Exported) található buyer\_id-ket és kiírja az így készült file-t az output könyvtárba.\
&nbsp;&nbsp;**csv_data.renew_buyer_ids** - A mappelés alapján kicseréli az input-csv-ben (Flat) található buyer\_id-ket és kiírja az így készült file-t az output könyvtárba.\
Egyéb fontos beállításokat, pl.: adatbázis-név, -jelszó, -port, vagy output file-ok adatait, a packages/settings-en belül tudod módosítani!

* Kommentezd ki programon belül ami nem kell, majd indítsd el a scriptet: `python run.py`
* Miután végzett, lépj ki a Python környezetből: `deactivate`
* Töröld a Contract-Output-ból az Exported.json-t és a Flat.csv-t: `rm ~/Path/To/Contract/Output/Directory/{Exported.json,Flat.csv}`
* Mozgasd az outputokat a data/output mappából a Contract-Output mappába: `mv ./data/output/output.* ~/Path/To/Contract/Output/Directory/`

Extended output előállítása
---------------------------

**Extra és Flat csv szükséges adatainak összevágása**
* Navigálj a dw\_flatten\_column\_exporter könyvtárba: `cd ~/Path/To/dw_flatten_column_exporter`
* Másold az extra és flat csv-ket (Contract és Tender külön-külön) ide (extra.csv és flat.csv néven!!!): `cp ~/Path/To/Output/Directory/{extra.csv,flat.csv} ./`
* Indítsd el az adatoknak megfelelő scriptet (python3!!! - itt nincs venv, meg kell mondani neki): `python3 index_contract.py`
* Másold az össze-mergelt csv-t egy tetszőleges temp-könyvtárba (még szükség lesz rájuk, de nem szerepelnek a végleges outputban): `cp ./output.csv ~/Path/To/Desired/Temp`

**Extended JSon és CSV előállítása**
* Navigálj a PostProcess Contract project data mappájába: `cd Path/To/PostProcess/project/KE_Contract/data`
* Másold az előző lépésben elkészített mergelt Contract csv-t ide source.csv néven: `cp ~/Path/To/Desired/Temp/Contract.csv ./source.csv`
* Másold a Contract-Output Exported JSon-jét ide source.json néven: `cp ~/Path/To/Contract/Output/Directory/Exported.json ./source.json`
* Lépj vissza a PostProcess root-könyvtárába: `cd ../../..`
* Indítsd el a Contract-okhoz tartozó scriptet: `./KE_C.sh`
* Készíts mappát az Extended-Outputnak, ha még nem tetted: `mkdir ~/Path/To/Extended/Output/Directory`
* Másold az így generált updated-source.csv-t és KE-extended-output.json-t az Extended-Output könyvtárba: `cp ./project/KE_Contract/data/{updated-source.csv,KE-extended-output.json} ~/Path/To/Extended/Output/Directory`
* Jelöld meg a csv-t Contractnak (átnevezés): `mv ~/Path/To/Extended/Output/Directory/updated-source.csv ~/Path/To/Extended/Output/Directory/Contract-updated-source.csv`
* JSon-t hasonlóan!
* Tenderrel is csináld meg ugyanezeket!

**Mergeld az outputokat**
* Menj a merger mappájába: `cd Path/To/merger`
* Indítsd el a Python környezetet: `source merger-venv/bin/activate`
* Másold a Contract és Tender updated-source-csv-it a csv mappába: `cp ~/Path/To/Extended/Output/Directory/{Contract-updated-source.csv,Tender-updated-source.csv} ./`
* Futtasd a programot (Mergeli a mappában található összes csv-t): `python index.py`
* Készíts egy mappát a Merged-Outputnak: `mkdir ~/Path/To/Merged/Output/Directory`
* Mozgasd a merged_csv.csv-t a Merged-Output könyvtárba: `cp ./merged_csv.csv ~/Path/To/Merged/Output/Directory`
* Flatten-Tool kimenetet hasonlóan!
* Lépj át a json mappába: `cd ../json`
* Másold ide az Extended json-öket: `cp ~/Path/To/Extended/Output/Directory/{Contract-KE-extended-output.json,Tender-KE-extended-output.json} ./`
* Futtasd a scriptet, aminek paraméterekként tudod átadni a json-öket: `./json_merger.sh Tender-KE-extended-output.json Contract-KE-extended-output.json`
* Mozgasd a merged.json-t a Merged-Output könyvtárba: `cp ./merged.json ~/Path/To/Merged/Output/Directory`
* Lépj ki a Python környezetből: `deactivate`


Python VENV-Tutorial
====================

VENV = Virtual Environment (Virtuális Környezet): Ide telepítjük azokat a package-eket, amik a működéshez elengedhetetlenek.\
Mellékelem az általam készített venv-eket a scriptek mellett, de lehet, hogy nálad nem indulnak rendeltetésszerűen ezért leírom, hogyan tudsz sajátot létrehozni.

1. Python3-al hozz létre egy környezetet: `python3 -m venv my_virtual_enviroment`
2. Aktiváld a környezetet: `source my_virtual_enviroment/bin/activate`
3. Telepítsd a dependencyket (ezeket a requirements.txt-ben írtam össze): `pip install -r requirements.txt`
4. Futtasd a programot: `python run.py`
5. Állítsd le a környezetet: `deactivate`

