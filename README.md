source: https://deepwiki.com/sandorkollar/AGDB

## Az AGDB áttekintése

Az AGDB egy Zig nyelven írt, nagy teljesítményű, állandó adatbázis-motor, amelyet a modern hardverekhez és a felhőalapú többfelhasználós környezetekhez terveztek. Réteges architektúrával rendelkezik, amely az alacsony szintű hardveres optimalizálásoktól (SIMD, GPU, NUMA) egészen egy olyan magas szintű felhőszolgáltatásig terjed, amely képes több ezer felhasználói munkaterhelés elszigetelésére Linux-alapú homokozótechnika segítségével.

A projekt átfogó eszközkészletet kínál, amely magában foglalja az alapvető motorkönyvtárat, az adatbázissal való közvetlen kommunikációra szolgáló parancssori felületet, valamint a felügyelt tárhelyszolgáltatáshoz szükséges felhőszervert.

### Rendszerfelépítés

Az AGDB kódbázisa három fő rétegre tagolódik:

1.   Core Engine (src/agdb.zig): Az alapvető tárolási és keresési logika, beleértve a tartós halmot, a WAL-alapú tranzakciókat és a hibrid keresési indexeket.
2.   Runtime (src/runtime.zig): Az a futtatási környezet, amely összehangolja az alapvető összetevőket, és kezeli a hardveres erőforrásokat, például a NUMA-csomópontokat és a GPU-kontextusokat.
3.   Cloud Layer (src/cloud/): Egy többfelhasználós felügyeleti rendszer, amely a hitelesítést, a kérések továbbítását és a felhasználói adatbázisok folyamatok szintjén történő elszigetelését kezeli.

### Alkatrészek közötti kapcsolat

Ez az ábra bemutatja, hogyan kapcsolódnak egymáshoz a főbb alrendszerek, összekapcsolva a magas szintű fogalmakat az azokhoz tartozó kódelemekkel.

### Főbb képességek

- Hibrid keresés: Az AGDB a hagyományos szövegkeresést (BM25) és a vektoros hasonlósági keresést ötvözi, így egységes keresési felületet biztosít.
- BM25: Fordított index alapú pontozás a kulcsszavak relevanciájának értékeléséhez
- Vektor: SIMD- és GPU-gyorsított hasonlósági keresés olyan mérőszámok felhasználásával, mint a koszinusz- és az euklideszi távolság

- Tartós tárolás: Egy egyedi PersistentHeap kezeli a memóriába leképezett fájlokat, összeomlásbiztos RelativePtr címzéssel és méretosztályon alapuló PersistentAllocator-ral
- ACID-tranzakciók: A Write-Ahead Log (WAL) és a TransactionManager szigorú konzisztenciát biztosít, és ahol lehetséges, a nagy párhuzamos feldolgozási teljesítmény érdekében a hardveres tranzakciós memóriát (HTM) használja
- Többfelhasználós rendszer: A felhőréteg Linux névterekkel és cgroupokkal szeparálja a bérlők munkaterheléseit a sandbox_runner folyamatokon belül, biztosítva ezzel az erőforrás-korlátozásokat és a biztonságot

### Tárhely és bináris fájlok

A projekt a Zig 0.14.0 eszközkészlet segítségével készült. A fordítási rendszer az ökoszisztéma különböző szerepeire szabott, többféle bináris fájlt állít elő:

| Bináris | Forráskód | Leírás |
| --- | --- | --- |
| agdb | src/cli_main.zig | Parancssori felület a helyi adatbázis-műveletekhez. |
| agdb-cloud | src/cloud_main.zig | A többfelhasználós API-t biztosító HTTP-kiszolgáló. |
| agdb-runtime | src/runtime_main.zig | Az önálló futtatómotor. |
| sandbox_runner | src/cloud/sandbox_runner.zig | A bérlői folyamatok elszigetelt végrehajtója. |
| agdb-wake-proxy | src/wake_proxy_main.zig | Életciklus-proxy a szunnyadó futásidejű környezetek aktiválásához. |

A könyvtárszerkezet és a bináris szerepkörök részletes leírását lásd

### Hardver és alacsony szintű architektúra

Az AGDB-t úgy tervezték, hogy kihasználja a modern szerverhardverek adta lehetőségeket. Tartalmaz speciális modulokat a NUMA-kompatibilis memóriakiosztáshoz, az alacsony késleltetésű hálózati kommunikációhoz szükséges RDMA-t, valamint a számításigényes keresési feladatok SIMD/GPU-alapú gyorsításához.

### Következő lépések

Ha az AGDB-projekt egyes területeit szeretné jobban megismerni, tekintse meg az alábbi aloldalakat:

- Útmutató a Zig-környezet beállításához, a projekt Makefile-on keresztüli lefordításához, valamint a felhőszolgáltatás deploy.sh segítségével történő telepítéséhez.
- Útmutató a forráskód felépítéséhez és a build.zig fájlban elérhető konfigurációs beállításokhoz.
- Az agdb könyvtárról szóló részletes dokumentáció, amely kiterjed a tárolásra, a tranzakciókra és a hibrid keresőmotorra.
- Az HTTP API, a bérlői nyilvántartás és a sandbox-szigetelési mechanizmus bemutatása.

## Első lépések: Készítés, futtatás és telepítés

Ez az oldal technikai útmutatást nyújt az AGDB fejlesztői környezet beállításához, a helyi fordítások futtatásához és az éles környezetbe történő telepítéshez. Az AGDB a Zig eszközkészletet és egy sor shell szkriptet használ a többkomponensű fordítások kezeléséhez, a környezet konfigurálásához és a systemd-be való integrációhoz.

### Eszközkészlet-követelmények

Az AGDB-hez a Zig 0.14.0 verziója szükséges. A fordítási rendszer úgy van kialakítva, hogy vagy a projekt gyökérkönyvtárában található helyi Zig-bináris fájlt használja, vagy a fordítási/telepítési folyamat során automatikusan letölti a megfelelő verziót a /tmp könyvtárba.

### Build Wrapper (build.sh)

A build.sh szkript biztonsági burkolatként működik, amely biztosítja, hogy a megfelelő Zig-verzió kerüljön használatra. Először ellenőrzi, hogy létezik-e helyi ./zig futtatható fájl, majd csak ezután tér vissza a rendszerkönyvtárhoz vagy a /tmp letöltési helyhez.

### Makefile-célok

A Makefile biztosítja a helyi fejlesztés és a fordítási folyamatok összehangolásának fő felületét.

| Cél | Parancs | Cél |
| --- | --- | --- |
| build | zig build -Doptimize=Debug | Az összes bináris fájlt hibakeresési szimbólumokkal és futásidejű biztonsági ellenőrzésekkel fordítja le  |
| release | zig build -Doptimize=ReleaseSafe -Dtarget=x86_64-linux-musl | Optimalizált, statikusan linkelt bináris fájlokat állít elő Linux számára  |
| serve | zig build ... && ./zig-out/bin/agdb-cloud | Összeállítja és azonnal elindítja az agdb-cloud szervert helyi környezetben  |
| telepítés | kiadás + ./deploy.sh | Készíti el a kiadási bináris fájlokat, és elindítja a termelési telepítési szkriptet  |
| clean | rm -rf .zig-cache zig-out | Eltávolítja a fordítási melléktermékeket és a gyorsítótárat  |

### Helyi végrehajtás: a Wake Proxy

Az életciklus-kezelő rendszer helyi teszteléséhez a run.sh parancsot használjuk az agdb-wake-proxy összeállításához és elindításához. Ez a komponens kezeli az alvó futási környezetek aktiválását.

### Adatáramlás: Helyi összeállítás és futtatás

A szkript ellenőrzi, hogy a Zig eszközkészlet rendelkezésre áll-e, összeállítja a proxyt a megadott rendszerleíró adatbázis- és adatgyökérkönyvtárakkal, valamint exportálja a felhőszolgáltatóval való integrációhoz szükséges környezeti változókat (pl. OVH API-kulcsokat)

Helyi összeállítási/futtatási logika

### Termelési környezetbe való telepítés

A deploy.sh szkript automatizálja a távoli Linux-kiszolgálóra történő élesítési folyamat teljes életciklusát. Kezelésére kerül a keresztkompilálás, az erőforrások szinkronizálása, a távoli könyvtárak előkészítése, a bináris fájlok terjesztése és a szolgáltatások kezelése.

### 1. Építkezés és az ingatlan előkészítése

A szkript először szinkronizálja a frontend index.html fájlt a felhőalapú forráskönyvtárba, majd a zig build parancsot futtatja a termelési környezetre jellemző paraméterekkel:

- Cél: x86_64-linux-musl statikus linkeléshez
- Optimalizálás: ReleaseSafe
- Fordítási állandók: Az AGDB_REGISTRY_PATH, AGDB_DATA_ROOT és sandbox_runner_path változókat közvetlenül beépíti a bináris fájlokba

### 2. Távoli infrastruktúra beállítása

Az SSH segítségével a szkript a következő környezetet állítja be a célszerveren

- Bináris fájlok: /opt/agdb/bin/
- Adat:/var/lib/agdb/tenants/
- Regisztrációs adatbázis: /var/lib/agdb/registry.agdb
- Futtató: /usr/lib/agdb/sandbox_runner
- Cgroups: Beállítja a cgroup v2-t a /sys/fs/cgroup/agdb könyvtárban úgy, hogy a memória-, CPU- és PID-vezérlők engedélyezve legyenek a bérlők elszigetelése érdekében

### 3. Szolgáltatások összehangolása

A telepítés két elsődleges systemd-szolgáltatást konfigurál:

| Szolgáltatás | Bináris | Szerepkör |
| --- | --- | --- |
| agdb-cloud | agdb-cloud | A fő többfelhasználós API-kiszolgáló. Emelt jogosultságokkal (CAP_SYS_ADMIN, CAP_SETUID stb.) fut a homokozók kezelése érdekében  |
| agdb-autoshutdown | agdb-autoshutdown | Az erőforrások visszanyerése érdekében az inaktív bérlői homokozókat leállító szolgáltatás  |

### 4. Az Nginx konfigurálása

A szkript egy olyan nginx.conf fájlt telepít, amely fordított proxyként működik, kezelve a Gzip-tömörítést és továbbítva a kéréseket az agdb-cloud belső portjára (7070)

Telepítési logika és kódelemek

## A tároló felépítése és a bináris csomag

Ez az oldal az AGDB-tárház fizikai felépítését, valamint a fordítási rendszer által létrehozott különböző bináris fájlok funkcionális szerepét ismerteti. Az AGDB a Zig fordítási rendszert (build.zig) használja az alacsony szintű végrehajtó motoroktól a magas szintű felhőkezelő proxy-kig terjedő komplex komponenscsomag kezelésére.

### A tárház felépítése

A kódtár egy fő könyvtárból és a különböző futtatási módokhoz tartozó több belépési pontból áll.

| Könyvtár / Fájl | Leírás |
| --- | --- |
| src/ | Az adatbázis-motor és a parancssori felület logikáját tartalmazza. |
| src/agdb.zig | A magkönyvtár gyökérmodulja, amely az összes főbb alrendszert exportálja. |
| src/cloud/ | Felhőalapú funkciókat tartalmaz, többek között a többfelhasználós szandboxolást és az IPC-t. |
| build.zig | A Zig build szkript, amely meghatározza a célokat, a függőségeket és a konfigurációt. |
| src/tests.zig | Integrációs tesztcsomag a teljes rendszerhez. |

### Fordítási konfiguráció

Az AGDB a build.zig fájlt használja arra, hogy a build_options modulon keresztül fordítási idejű állandókat illesszen be a bináris fájlokba. Ezek az opciók határozzák meg a kritikus fájlrendszer-útvonalakat és a telepítési helyeket.

- AGDB_REGISTRY_PATH: A többfelhasználós regisztrációs adatbázis helye (Alapértelmezett: /var/lib/agdb/registry.agdb).
- AGDB_DATA_ROOT: A bérlőspecifikus adatok tárolási gyökérkönyvtára (Alapértelmezett: /var/lib/agdb/tenants).
- sandbox_runner_path: A sandbox_runner bináris fájl abszolút telepítési útvonala, amelyet az agdb-cloud használ a sandboxok elindításához.

Források: 71

### Binary Suite

A fordítási rendszer hat különböző bináris fájlt hoz létre, amelyek mindegyike sajátos szerepet tölt be az adatbázis életciklusában.

### 1. agdb (parancssori felület)

A helyi adatbázis-példányokkal való kommunikációra szolgáló elsődleges parancssori felület. A CLI-argumentumokat közvetlenül az agdb.cli.run parancshoz rendeli.

- Belépési pont: src/cli_main.zig
- Logika: Elemezi az argumentumokat, és a magkönyvtárból hívja meg a CLI modult.

### 2. agdb-runtime (végrehajtó motor)

Egy önálló végrehajtó motor, amelyet egy adott adatbázis-példány (heap és WAL) felügyeletére használnak a felhőalapú környezeten kívül.

- Belépési pont: src/runtime_main.zig
- Fő funkció: agdb.Runtime.init
- Beállítások: A tartós környezet konfigurálásához a --heap, --wal, --snapshots és --size paramétereket fogadja el.

### 3. agdb-cloud (szerver)

A többbérlős vezérlőréteg. Ez kezeli a bérlői nyilvántartást, hitelesíti az API-kulcsokat, és irányítja a bérlői tesztkörnyezetek életciklusát.

- Belépési pont: src/cloud_main.zig
- Főbb összetevők: registry.Registry, process_table.ProcessTable és http_server.CloudServer.
- Működés: Létrehoz egy diszpécser szálat az IPC kezeléséhez, és elindítja a HTTP-kiszolgálót a 7070-es porton (beállítható az AGDB_CLOUD_PORT változóval).

### 4. sandbox_runner (izolált végrehajtó)

Egy speciális, statikusan linkelt bináris fájl, amelyet Linux-névtérben való futtatásra terveztek. Ez az egyetlen bináris fájl, amely a felhőbeli bérlő számára a tényleges adatbázis-műveleteket végzi.

- Belépési pont: src/cloud/sandbox_runner.zig
- Elszigetelés: Szigorú seccomp-szűrőt valósít meg (amely csupán ~80 rendszerhívást engedélyez), és létrehoz egy tmpfs-alapú chroot-környezetet, amelyben a rendszerkönyvtárak bind-mountolással vannak csatlakoztatva.
- GPU-integráció: Olyan kerneleit regisztrálja, mint például a cosineSimilarityKernel, a vektoros keresés felgyorsítása érdekében.

### 5. agdb-wake-proxy (Életciklus-proxy)

Egy könnyűsúlyú proxy, amelyet alacsony erőforrásigényű, „állandóan aktív” csomópontokon való futtatásra terveztek. A proxy elfogja az alvó állapotban lévő AGDB felhőalapú példányra érkező kéréseket, és ébresztő jelet indít el (pl. az OVH Cloud API-n keresztül).

- Belépési pont: src/wake_proxy_main.zig
- Viselkedés: 503-as „Szolgáltatás nem elérhető” hibakódot ad vissza egy egyedi „Waking Up” nevű HTML-oldallal, miközben a fő szerver /v1/health végpontját lekérdezi.

### 6. agdb-autoshutdown (Tétlenségi figyelő)

Egy segédprogram, amely figyelemmel kíséri a rendszer tevékenységét, és inaktivitás esetén a gazdagép leállításával csökkenti a költségeket.

- Belépési pont: src/autoshutdown_main.zig
- Működés: Elemezi a /var/log/nginx/access.log fájlt, hogy meghatározza az utolsó kérés időpontját. Ha az inaktivitás időtartama eléri vagy meghaladja az IDLE_SECONDS értéket (alapértelmezés szerint 15 perc), akkor végrehajtja a sudo systemctl poweroff parancsot.

### Rendszer-kölcsönhatási diagramok

### Komponensek összehangolása és IPC

Ez az ábra bemutatja, hogyan működnek együtt a különböző rendszerelemek, amikor egy kérés beérkezik a felhőrendszerbe.

## Felhőalapú kérelemfolyamat és bináris interakció

### Konfigurációs adatáramlás létrehozása

Ez az ábra bemutatja, hogyan kerülnek át a fordítási beállítások a felhasználótól a végső, lefordított elemekbe.

## Beépítési opciók betöltési útvonala

Források: 71

### A bináris tulajdonságok összefoglalása

| Bináris | Kapcsolódás | Cél operációs rendszer | Fő rendeltetés |
| --- | --- | --- | --- |
| agdb | Dinamikus | Natív | Helyi kezelés |
| agdb-runtime | Dinamikus | Natív | Adatbázis-végrehajtás |
| agdb-cloud | Dinamikus | Natív | Többfelhasználós vezérlés |
| sandbox_runner | Statikus | Linux (musl) | Elszigetelt bérlői terhelés |
| agdb-wake-proxy | Dinamikus | Natív | Hidegindítás-kezelés |
| agdb-autoshutdown | Dinamikus | Linux | Költségoptimalizálás |

Források: 148

## Alapvető adatbázis-motor

Az agdb könyvtár, amelyet a  fájlban definiáltak, az adatbázis-motor központi irányító központjaként szolgál. Egy többrétegű architektúrát foglal magában, amelyet nagy teljesítményű adatmentés, az ACID-elvek betartása és fejlett hibrid keresési funkciók biztosítására terveztek. A motor összeköti az alacsony szintű hardveres optimalizációkat (SIMD, GPU, NUMA) a magas szintű adatstruktúrákkal és keresési algoritmusokkal.

### Rendszerfelépítés

Az adatbázis-motor egy vertikális rétegekből álló felépítésű, amelyben minden réteg absztrakciókat biztosít a fölötte lévő réteg számára. Az alapszinten a rendszer a hardverrel és a fájlrendszerrel kommunikál, míg a legfelső réteg az alkalmazások számára biztosítja az adatbázis- és a PersistentStore-API-kat.

### A motor alkatrészeinek hierarchiája

Az alábbi ábra bemutatja, hogy a magas szintű Database struktúra hogyan kapcsolódik az alapul szolgáló kódelemekhez és tároló alrendszerekhez.

Adatbázis-összetevők térképe

### Főbb alrendszerek

A motor öt fő alrendszerből áll, amelyek mindegyikét külön aloldalak tárgyalják részletesen.

### 2.1 Állandó tárolóréteg

Az AGDB alapját a PersistentHeap képezi, amely a memóriába leképezett fájlokat kezeli és biztosítja a módosított oldalak nyomon követését. Az eltolásalapú címzéshez a PersistentPtr és a RelativePtr típusokat használja, így biztosítva, hogy a mutatók a folyamat újraindításakor is érvényben maradjanak. A memóriakiosztást ebben a halomban a PersistentAllocator kezeli, amely szlábkiosztást és hardverspecifikus utasításokat, például clwb-t és sfence-t használ a cache-sorok állandóságának biztosítására.

További részletekért lásd

### 2.2 Előreírási napló és tranzakciókezelés

Az AGDB a WAL (Write-Ahead Log) és a TransactionManager kombinációjával biztosítja az ACID tulajdonságokat. A TransactionManager a Hardware Transactional Memory (HTM) és a SeqLock-alapú pillanatképek segítségével támogatja az optimista párhuzamos feldolgozást. Rendszerleállás esetén a RecoveryEngine elvégzi az elemzést, valamint a redo/undo fázisokat, hogy az adatbázist konzisztens állapotba állítsa vissza.

További részletekért lásd

### 2.3 Kulcs-érték tárolók és állandó adatstruktúrák

A KvStore egy kizárólag hozzáfűzésre alkalmas naplóformátumot biztosít, memóriában tárolt hash-indexszel a gyors keresések érdekében. Ezen felül a PersistentStore API lehetővé teszi komplex perzisztens típusok, például térképek és tömbök kezelését a Handle<T> életciklus használatával. A Database struktúra ezeket a tárolókat használja belső állapotának perzisztálására, beleértve a next_id számlálót és az index metadatákat is

További részletekért lásd

### 2.4 Keresési indexek: BM25, vektoros és hibrid keresés

A motor támogatja a teljes szöveges keresést a Bm25Index segítségével, valamint a hasonlósági keresést a VectorIndex segítségével. Ezek a Database.searchHybrid algoritmusban kombinálhatók, így a rendszer mind a szemantikai, mind a kulcsszó-alapú relevancia alapján rangsorolt eredményeket ad. A VectorIndex többféle távolságmérőt támogat, és a GPUContext segítségével a nagy számítási terhelést a GPU-ra tudja átterhelni.

További részletekért lásd

### 2.5 Memóriakezelés és szemétgyűjtés

Az alapvető memóriakiosztáson túl az AGDB egy RefCountGC-t alkalmaz a tartós objektumok automatikus memóriavisszanyerésére. Ez cikuszáró mechanizmussal és kötegelt műveletekkel rendelkezik a rendszerterhelés minimalizálása érdekében. A rendszer emellett egy AdaptiveAllocator-t (JIT-allokátort) is használ a memóriahasználati minták futásidő alatti dinamikus optimalizálására.

További részletekért lásd

### Alapvető adatáramlás

A Database struktúra a fejlesztők számára az elsődleges belépési pontként szolgál. Amikor egy Database-t megnyitnak, az egy megadott könyvtárból inicializálja belső indexeit és kulcs-érték tárolóját.

Az adatbázis inicializálásának folyamata

### Műszaki összefoglaló táblázat

| Komponens | Kód | Fő feladatkör |
| --- | --- | --- |
| Orchestrator | Adatbázis | Magas szintű API CRUD-műveletekhez és kereséshez |
| Futási idő | Futási idő | Hardver- és tárolóeszközök életciklus-kezelése |
| Tárolás | PersistentHeap | MMap-kezelés és oldalperzisztencia |
| Naplózás | WAL | Tartósság a Write-Ahead Logging segítségével |
| Indexelés | Bm25Index | Fordított index szövegek értékeléséhez |
| Vektorok | Vektorindex | Hasonlósági keresés és beágyazások |
| Biztonság | RecoveryEngine | Összeomlás utáni helyreállítás és integritás |

## Állandó tárolóréteg

A tartós tárolási réteg az AGDB tartóssági motorjának alapját képezi, amely nagy teljesítményű, összeomlásbiztos absztrakciót biztosít a memóriába leképezett fájlok felett. Ez kezeli az adatok fizikai elrendezését a lemezen, gondoskodik a gyorsítótárral összehangolt kiírásról a tartós adathordozókra, valamint egy olyan kifinomult allokátort biztosít, amelyet kifejezetten az alacsony késleltetésű adatbázis-terhelésekhez terveztek.

### PersistentHeap és memóriatérképzés

A PersistentHeap az adatbázis állapotát tároló memóriába leképezett fájl kezelésének elsődleges felülete. Az mmap parancsot (Linux/macOS rendszereken) vagy a CreateFileMapping függvényt (Windows rendszereken) használja az adatbázis-fájlnak a folyamat virtuális címtérébe történő leképezéséhez.

### Hardveres gyorsítótár-kezelés

Annak biztosítására, hogy az adatok valóban a tartós adathordozókra (pl. Optane DC Persistent Memory vagy SSD-k) kerüljenek írásra, az AGDB architektúraspecifikus gyorsítótár-ürítési utasításokat használ. A rendszer fordítási és futási időben felismeri a hardver képességeit, hogy kiválaszthassa a leghatékonyabb megoldást

- clwb (Cache Line Write Back): A gyorsítótár-sor visszaírása annak érvénytelenítése nélkül, minimalizálva ezzel a késleltetést a későbbi olvasási műveleteknél
- clflushopt: Optimalizált cache-sor-kiürítés
- sfence: Biztosítja, hogy az összes korábbi tárolás és kiürítés globálisan látható legyen, mielőtt továbbhaladna
- movnti: Nem időfüggő tárolók esetében használják, hogy nagy pufferek írásakor teljesen megkerüljék a CPU-gyorsítótárat.

### A „Dirty Page” nyomon követése

A PersistentHeap egy dirty_pages bitkészletet tart fenn annak nyomon követésére, hogy mely memórioldalakat módosították. Ez lehetővé teszi az inkrementális szinkronizálást az msync parancs vagy a kézi gyorsítótár-ürítés segítségével, ami jelentősen csökkenti az I/O-terhelést a tranzakciók véglegesítése során.

### A halom életciklusának ábrája

Az alábbi ábra bemutatja a fizikai fájl, a memóriatérkép és az irányítási struktúrák közötti kapcsolatot.

A PersistentHeap architektúra

### Tartós allokáció

A PersistentAllocator kezeli a memóriát a PersistentHeap-en belül. Ez egy szelvényalapú allokátor, amely méretosztályokat használ a töredezettség minimalizálására és a kis objektumok esetében O(1) allokációs idő biztosítására

### Méretosztály szerinti tábla-elosztás

A legfeljebb 4096 bájt méretű objektumokat szeletek kezelik. Minden szeletet a SizeClass alapján egyenlő méretű részekre osztanak.

- Kis objektumok: 32 méretosztályt használ, amelyek mérete exponenciálisan, körülbelül 1,25-szeresére növekszik
- Nagy objektumok: A méretosztály küszöbértékét meghaladó objektumok közvetlenül a heapből kerülnek kiosztásra a large_free_list-ben, „first-fit” vagy „best-fit” stratégia alkalmazásával

### MPSC ingyenes várólisták

A nagy párhuzamosságú memóriakibocsátás támogatására az allokátor MPSC (Multi-Producer Single-Consumer) típusú felszabadítási sorokat (mpsc_lanes) valósít meg. Ez lehetővé teszi a szálak számára, hogy a globális allokátor zárolásának megszerzése nélkül „felszabadítsák” a memóriát azáltal, hogy a címet egy sávba tolják. A méretosztály elsődleges tulajdonosa később kiüríti ezeket a sorokat a memória visszanyerése érdekében

### Címzés: PersistentPtr és RelativePtr

A hagyományos mutatók a folyamat újraindításakor vagy abban az esetben érvényüket vesztik, ha a fájl egy másik alapcímre van leképezve. Az AGDB ezt az eltolásalapú címzéssel oldja meg.

### Tartós mutató

A PersistentPtr egy tárolási hely globálisan egyedi azonosítója. A következő elemekből áll:

1.   pool_uuid (u128): A PersistentHeap egyedi azonosítója
2.   eltolás (u64): A halom alapcímétől számított bájteltolás

### RelativePtr

A RelativePtr a tartós adatstruktúrákban használt mutatók optimalizált változata. Tartalmaz egy címkézett mezőt, amely támogatja az inline értékeket. Ha az INLINE_FLAG be van állítva, a mutató nem egy eltolásra mutat, hanem egy kis, 15 bites értéket tartalmaz közvetlenül a mutató struktúrájában, ezzel megspórolva egy memóriadereferenciálást kis egész számok vagy enumok esetén.

| Mező | Típus | Leírás |
| --- | --- | --- |
| eltolás | u64 | Bájteltolás az alapcímtől |
| pool_uuid_low | u64 | A pool-azonosító alsó 64 bitje |
| pool_uuid_high | u64 | A pool-azonosító felső 64 bitje |
| címkézett | u32 | Címke bitek és beágyazott jelző |

### Adatformátumok

A HeapHeader az AGDB-fájlok első 256 bájtja, amely a helyreállításhoz és az érvényesítéshez elengedhetetlenül szükséges metaadatokat tartalmazza:

- Magic & Verzió: ZIGPHEAP 1. verzió
- UUID: Az adatbázis-pool egyedi azonosítója
- Ellenőrző összeg: a fejléc CRC32c-értéke a sérülések felismerése érdekében
- Tranzakciós azonosító: Az utolsó sikeresen végrehajtott tranzakció azonosítója

Minden allokált objektumot egy ObjectHeader előz meg

- magic:0xDEADBEEF
- ref_count: A RefCountGC használja az automatikus memóriakezeléshez
- jelzők: Jelezi, hogy az objektum felszabadult-e, rögzítve van-e, vagy tömb-e

Objektumok memóriában való elrendezése

### Öblítés és szinkronizálás életciklusa

Az írási művelet végrehajtása szigorú sorrendet követ, hogy biztosítsa a rendszer összeomlás esetén is fennmaradó konzisztenciáját:

1.   Írás: Az adatok a memóriába leképezett területre kerülnek.
2.   Mark Dirty: A dirty_pages mezőben a megfelelő bitek be vannak jelölve
3.   flushRange: A rendszer a cache_flush függvényt hívja meg (a clwb/clflushopt használatával) az adott módosított gyorsítótár-sorok esetében
4.   memória-kerítés: A memória-kerítés biztosítja, hogy a kiürítések teljes mértékben végbemenjenek
5.   WAL-véglegesítés: A tranzakciókezelő egy véglegesítési bejegyzést ír a Write-Ahead Logba (lásd a 2.2. szakaszt).
6.   msync: A PersistentHeap rendszeres időközönként meghívja az msync parancsot az operációs rendszer oldalkészletének és a fizikai lemez közötti szinkronizáláshoz

A parancsok végrehajtásának folyamata

## Előreírási napló és tranzakciókezelés

Az AGDB Write-Ahead Log (WAL) és tranzakciókezelő rendszerei teljes ACID-garanciát (atomicitás, konzisztencia, izoláció, tartósság) biztosítanak minden adatbázis-művelet számára. A rendszer egy fizikai redo/undo naplót használ, amelyhez egy többverziós tranzakciókezelő társul, amely olyan hardverfunkciókat használ ki, mint a Hardware Transactional Memory (HTM) és az io_uring a nagy teljesítményű commit-csatornákhoz.

### Előreírási napló (WAL)

A WAL egy memóriába leképezett naplófájl, amely rögzíti az összes állapotváltozást, mielőtt azok a PersistentHeap-re kerülnének. Körkörös puffer szerkezetet használ a naplóbejegyzések kezeléséhez, és támogatja az aszinkron, kötegelt és csővezetékes rögzítési stratégiákat.

### A WAL felépítése és rekordjai

A WAL-fájl egy WALHeader-rel kezdődik, amely nyomon követi a head_offset, a tail_offset és a last_checkpoint LSN értékeket. A napló minden bejegyzése egy WALRecord, amely tartalmaz egy RecordType-ot (pl. begin, commit, write, allocate, free), egy tranzakciós azonosítót, valamint CRC32c ellenőrző összegeket mind a rekord metaadatokra, mind a kapcsolódó adatpakettre vonatkozóan.

### Elkötelezettségi stratégiák

Az AGDB több I/O-útvonalat támogat a WAL-tároláshoz:

1.   Szinkronizálás/Tömeges feldolgozás: A rekordok hozzáadódnak, majd az msync vagy az fdatasync parancs segítségével azonnal elmentésre kerülnek.
2.   Pipelined: A WALIOUringWriter segítségével kihasználja a Linux io_uring funkcióját, így egyetlen rendszerhívás keretében több naplóbejegyzés is elküldhető a kerneltnek anélkül, hogy a tranzakciókezelő működése megakadna.
3.   Fuzzy ellenőrzőpontok: Az ellenőrzőpont-rekord lehetővé teszi a napló rövidítését. A WALHeader tárolja a last_checkpoint LSN-t a helyreállítási idő korlátozása érdekében.

### Kód-entitás-térkép: A WAL belső működése

Az alábbi ábra bemutatja a WAL logikai felépítésének megfelelő implementációs struktúrákat és típusokat.

Források:   8

### Tranzakciókezelés

A TransactionManager koordinálja a tranzakciók életciklusát, biztosítva az elszigeteltséget és az atomikusságot. Hibrid végrehajtási útvonalat támogat, amely a gyors útvonal érvényesítéséhez a hardveres tranzakciós memóriát (HTM), az állapotpillanatképek nyomon követéséhez pedig a SeqLock-alapú mechanizmust használja.

### A tranzakció életciklusa

1.   Kezdés: A Transaction objektumot egy egyedi azonosítóval és egy hozzá tartozó wal_tx-szel inicializálják.
2.   Műveletek nyomon követése: Ahogy a tranzakció olvasási és írási műveleteket hajt végre, feltölti a read_set és a write_set halmazokat
3.   Ütközésfelismerés: A rögzítés előtt a rendszergazda ellenőrzi a következőket:
- Írás-írás: Két tranzakció próbálja módosítani ugyanazt az eltolást.
- Olvasás-írás / Írás-olvasás: A sorba rendezhetőség biztosítása azáltal, hogy ellenőrizzük, hogy az olvasási halmazokat nem módosították-e párhuzamosan végrehajtott, már véglegesített tranzakciók

4.   Érvényesítés (HTM Fast-Path): Amennyiben a CPU támogatja, az AGDB az htm.zig segítségével egyetlen hardveres tranzakció keretében optimisztikusan érvényesíti a tranzakció olvasási és írási hash-értékeit.
5.   Véglegesítés/visszavonás: Siker esetén a rendszer véglegesítési bejegyzést ír a WAL-ba. Hiba esetén a visszavonási fázis a WAL visszavonási adatait használja a módosítások visszaállításához.

### Tranzakciós állapot és párhuzamosság

A tranzakciók a RegisterResidentTxState típust használják, amely pontosan 64 bájt (egy cache-sor) méretű, a hamis megosztás minimalizálása érdekében. Az állapotváltásokat SeqLock védi, hogy a tranzakciós metaadatokhoz nagy párhuzamosságú, olvasási oldali hozzáférés legyen lehetséges.

### Helyreállító motor

A RecoveryEngine feladata, hogy az adatbázist összefüggő állapotba állítsa vissza egy összeomlás után. A futásidejű inicializálás során automatikusan elindul, ha a PersistentHeap fejléc dirty jelzője be van kapcsolva.

### A felépülés szakaszai

A helyreállítási folyamat az ARIES-típusú protokollt követi, amely négy különálló szakaszból áll:

| Fázis | Cél | Végrehajtás |
| --- | --- | --- |
| Elemzés | Átvizsgálja a WAL-t az utolsó ellenőrzőponttól kezdve, hogy azonosítsa a véglegesített és a befejezetlen tranzakciókat. | runAnalysisPhase |
| Redo | Az LSN-sorrendben újra végrehajtja a véglegesített tranzakciók összes műveletét, hogy azok biztosan megjelenjenek a heapben. | runRedoPhase |
| Visszavonás | A visszavonási adatok felhasználásával visszaállítja a befejezetlen (aktív/előkészített) tranzakciók műveleteit, az LSN-sorozat fordított sorrendjében. | runUndoPhase |
| Finalize | Törli a heap „dirty” jelölését, és frissíti a WAL-fejléceket. | finalizeRecovery |

### Ütközésszimuláció

A helyreállítási logika megbízhatóságának biztosítása érdekében az AGDB tartalmaz egy CrashSimulator modult. Ez lehetővé teszi a fejlesztők számára, hogy meghatározott pontokon (pl. a redo vagy undo fázisok során) hibákat szimuláljanak, így ellenőrizve, hogy a helyreállítás idempotens-e, és megfelelően kezeli-e a részleges helyreállítási kísérleteket.

### Kód-entitás térkép: Helyreállítási folyamat

Ez az ábra bemutatja, hogyan működik együtt a RecoveryEngine a WAL-lal és a Heap-pel a rendszer újraindításakor.

### Épség és javítás

A szokásos helyreállításon túl az AGDB egy HeapRepair segédprogramot is biztosít. Ez a komponens az adatbázis állapotának alapos vizsgálatát végzi el, többek között:

- Ellenőrzőösszeg-ellenőrzés: az AllocatorMetadata és az ObjectHeader ellenőrzőösszegének ellenőrzése
- A szabadlisták újjáépítése: A halom átvizsgálása az allokátor szabadlistájának helyreállítása érdekében, amennyiben a metaadatok megsérültek
- Objektumok jelölése: A szivárgó vagy elhagyott objektumok azonosítása és jelölése

## Kulcs-érték tároló és állandó adatstruktúrák API

Ez az oldal a KvStore-t és a magas szintű PersistentStore API-t ismerteti. Ezek a komponensek biztosítják az AGDB alapvető tárolási absztrakcióit, átlépve a nyers, kizárólag hozzáfűzésre alkalmas naplófájlokról a kifinomult, összeomlásbiztos, állandó adatstruktúrákra, mint például a térképek és a tömbök.

### KvStore: Csak hozzáfűzéses napló és indexelés

A KvStore a klasszikus, naplóstruktúrájú, szomszédos egyesítéses mintát valósítja meg. A tartós tárolás érdekében kizárólag hozzáfűzésre alkalmas fájlformátumot, a gyors keresés érdekében pedig memóriában tárolt hash-indexet használ.

### Adatelrendezés és integritás

Minden írási művelet (put vagy delete) a fájl végéhez kerül hozzáfűzve egy RecordHeader formájában, amelyet a kulcs és az érték bájtok követnek

- Integritás: Minden rekord tartalmaz egy CRC32 ellenőrző összeget, amelyet a művelet típusa, a kulcs, az érték és az időbélyeg alapján számítottak ki
- Hibahelyreállítás: Induláskor az alkalmazás végrehajtja a readAndRepairHeader és a replayLog parancsokat. Ellenőrzi minden rekord CRC-értékét és mágikus számát, és a fájl sérülésének első jele esetén levágja azt, hogy helyreállítsa a konzisztens állapotot.
- Memóriában tárolt index: Az indextábla olyan Entry típusú struktúrákat tárol, amelyek tartalmazzák az adott kulcs legfrissebb verziójának fájlpozícióját és hosszát

### Tömörítési folyamat

A rekordok frissítése vagy törlése során „halott bájtok” halmozódnak fel a fájlban. A KvStore nyomon követi ezt a töredezettséget. Amikor egy küszöbérték elérése esetén egy tömörítési folyamat (amely elvileg hasonló a GC-hez) az indexből kizárólag az aktív bejegyzéseket írja át egy új, összefüggő fájlba, így helyet szabadítva fel.

### Adatáramlás: Helyezés művelet

Az alábbi ábra bemutatja egy put kérés útját a KvStore logikán keresztül.

KvStore elhelyezési logika

### PersistentStore API

A PersistentStore (és a hozzá tartozó Handle<T>) magas szintű, objektumorientált felületet biztosít a tartós memóriatömbhöz. Elvonja a figyelmet a kézi eltoláskezelés és a tranzakciók rögzítésének bonyolultságairól.

### A kezelő életciklusa

A Handle(comptime T) az állandó objektumokkal való interakció elsődleges mechanizmusa. Ez kezeli az átmenetet az állandó tároló és a natív Zig-mutatók között.

| Funkció | Leírás |
| --- | --- |
| init | Létrehoz egy PersistentPtr-hez tartozó hivatkozót |
| get | A natív T típusra mutató állandó mutatót értelmezi ki |
| szerkesztés | A kezelőt .write módba állítja, és rögzíti a műveletet a Tranzakciókezelőben  |
| commit | A módosított natív memóriát visszaírja az állandó memóriatömbbe  |

### Tartós adatstruktúrák

Az AGDB számos beépített adatstruktúrát biztosít, amelyek teljes egészükben a tartós halomban találhatók:

1.   PersistentArray(T): Bővíthető tömb, amely a reallocate metódussal kezeli a kapacitást
2.   PersistentMap(K, V): Hash-térkép megvalósítás tartós tárolóblokkok és bejegyzésláncok felhasználásával
3.   ResidentObjectTable: Nyomon követi a memóriába jelenleg betöltött objektumokat, hogy elkerülje a felesleges feloldásokat, és kezelje a RefCountGC-integrációt.

Rendszerelemek leképezése: API és tároló

### Adatok állandósága és sémák

Az adatbázisréteg a KvStore-t használja a magas szintű Record objektumok tárolására. Ezek a rekordok a felhasználó által tárolt tényleges adatelemeket (dokumentumok, beszélgetések stb.) képviselik.

### Felvételi formátum

A RecordWriter a Record-ot bináris formátumba konvertálja

- Fejléc: Tartalmazza a RECORD_MAGIC (0x52454331) értéket, a RecordKind mezőt és a rekord azonosítóját
- Metaadatok: tartalmazza a created_at_us és az updated_at_us időbélyegeket
- Adatcsomag: Változó hosszúságú címkék, a fő szövegrész, valamint egy opcionális vektorbeágyazás

### Séma-nyilvántartás

A SchemaRegistry kezeli a tartós struktúrák felépítését. Lehetővé teszi a rendszer számára az adatok integritásának ellenőrzését, valamint a migrációk végrehajtását, amikor a struktúra-definíciók megváltoznak.

- Regisztráció: a registerSchema egy Zig típust fogad be, és létrehoz egy StructInfo objektumot, amely tartalmazza a mezők elhelyezkedését, méretét és az ellenőrző összeget
- Ellenőrzés: a validateObject biztosítja, hogy egy memóriablokk megfeleljen a várt sémakövetelményeknek
- Migráció: A nyilvántartó MigrationFn visszahívásokat hajthat végre az adatok régi sémaváltozatból újba történő átalakításához

A rekordok sorosításának folyamata

### A legfontosabb szervezetek áttekintése

| Entitás | Fájl | Szerepkör |
| --- | --- | --- |
| KvStore |  | Tartós, kizárólag hozzáfűzésre alkalmas kulcs-érték tároló CRC-védelemmel. |
| Handle(T) |  | RAII-stílusú burkoló a tartós objektumok életciklusához. |
| PersistentArray |  | A tartós halomban tárolt, dinamikusan méretezett tömb. |
| Rekord |  | Az adatbázisréteg elsődleges adateleme. |
| SchemaRegistry |  | A tartós objektumok típus-metadatáit és migrációit kezeli. |

Források: 15 111 12 62

## Keresési indexek: BM25, vektoros és hibrid keresés

Az AGDB egy multimodális keresőmotort biztosít, amely ötvözi a hagyományos teljes szöveges keresést, a magas dimenziójú vektoros hasonlósági keresést és egy hibrid rangsorolási rendszert. Ezeket a funkciókat a Database osztály által kezelt állandó indexek formájában valósítják meg, kihasználva mind a CPU-ra optimalizált algoritmusokat, mind pedig – amennyiben rendelkezésre áll – a GPU-gyorsítást.

### BM25 teljes szövegű index

A Bm25Index a „Best Matching 25” rangsorolási funkciót valósítja meg, amely megbízható, kulcsszóalapú keresést biztosít. Egy egyedi tokenizálóval integrálva a nyers szöveget kereshető fordított indexsé alakítja át.

### Tokenizálás és fordított indexelés

Az indexelési folyamat a tokenizer.zig modullal kezdődik. A tokenize függvény a nyers bájtokat TokenList-té alakítja, támogatva az UTF-8 dekódolást, a kisbetűs átalakítást és az alfanumerikus szűrést

Amikor egy dokumentumot a Bm25Index.addDocument metódussal adnak hozzá

1.   A szöveget különálló kifejezésekre bontják.
2.   A dokumentumra kiszámítják a kifejezésgyakoriságokat (TF)
3.   A Posting típusú struktúra (amely tartalmazza a doc_id és a tf mezőket) hozzáadódik a postings térképhez, amely a token 64 bites hash-értékét használja kulcsként
4.   A dokumentumok hossza és a teljes szószám frissítésre került a BM25-normalizálás támogatásához

### Pontszámítási algoritmus

A keresési funkció a pontszámokat a szokásos BM25-képlet alapján számítja ki. Minden keresési kifejezésre kiszámítja az inverz dokumentumgyakoriságot (IDF): $IDF(q_i) = \ln(\frac{N - n(q_i) + 0.5}{n(q_i) + 0.5} + 1)$ Ahol $N$ a total_docs, és $n(q_i)$ a kifejezést tartalmazó dokumentumok száma

### Vektorindex

A VectorIndex hasonlósági keresést biztosít nagy dimenziójú ágyazásokhoz. Többféle távolságmérőt támogat, és SIMD-re és GPU-ra optimalizált brute-force módszert alkalmaz.

### Mérőszámok és tárolás

Az index a Distance enum-ban meghatározott négy fő távolságmérőt támogatja

- Kozinus-hasonlóság: A vektorok közötti szög kozinuszát méri
- Belső szorzat: Szabványos pontszorzásos hasonlóság
- Euklideszi távolság: $L_2$-normás távolság
- Manhattan-távolság: $L_1$-normás távolság

A vektorokat VectorEntry típusú struktúrákban tárolják, amelyek tartalmazzák az előre kiszámított normát a koszinusz-hasonlóság számításainak felgyorsítása érdekében

### Jellemző-hash (hashEmbed)

Azokban az esetekben, amikor előre kiszámított beágyazások nem állnak rendelkezésre, az AGDB a hashEmbed függvényt használja. Ez a függvény a szöveg tokenizálásával végzi el a jellemzők hash-elését (az úgynevezett „hashing trick” módszert), majd a tokenek hash-értékeit gördülő hash segítségével egy fix méretű vektortérbe képezi le.

### Top-K-keresés

Az adatlekérdezést egy MaxHeap kezeli. Keresés során az index végigfut az összes bejegyzésen, kiszámítja a pontszámot, és a legjobb K eredményt tárolja a halomban, hogy biztosítsa az $O(N \log K)$ időbeli komplexitást.

### Hibrid keresés és újrarangsorolás

Az AGDB a Database.searchHybrid modulban egy alfa-súlyozott egyesítési algoritmus segítségével egyesíti a BM25 és a Vector keresési eredményeit

### Pontszámok összevonása

A hibrid keresés egyszerre hajt végre BM25-keresést és vektoros keresést, majd egyesíti az eredményeket:

- Alfa-súlyozás: Az $\alpha$ paraméter (alapértelmezett érték: 0,5) egyensúlyba hozza a BM25-pontszámok és a vektoros pontszámok hozzájárulását
- Normalizálás: A két index pontszámait az összevonás előtt a [0, 1] tartományba normalizáljuk az összeegyeztethetőség biztosítása érdekében

### RankIndex és Ranker

A RankIndex egy további átsorolási réteget biztosít. A következőket használja:

1.   SSI (szekvenciaszegmens-index): A tokenek szekvenciáinak nyomon követése a távolságot figyelembe vevő értékeléshez
2.   MinHash: Támogatja a dokumentumok közötti Jaccard-hasonlóság becslését a minHashSignature használatával
3.   N-gram súlyok: A rangsoroló algoritmus az újrarangsorolási fázisban csökkenő súlyokat alkalmaz a magasabb rendű n-gramokra

### Hibrid keresési adatáramlás

Az alábbi ábra bemutatja a természetes nyelvű lekérdezéstől a rangsorolt QueryResult-objektumok halmazáig vezető folyamatot.

## Hibrid keresési folyamat

### GPU-gyorsítás

Az AGDB a vektorok közötti hasonlóság számításait a GPU-ra ruházza át, ha az adatkészlet mérete meghaladja a gpu_search_threshold értéket

### GPU-útvonal végrehajtása

A Database.searchVector metódus meghívásakor a rendszer ellenőrzi, hogy rendelkezésre áll-e GPUContext. Ha az elemek száma elég nagy, akkor a gpu_mod.cosine_similarity_batch (vagy hasonló kernelek) metódust hívja meg.

## Vektoros keresési diszpécserlogika

### Sorozatkezelés és állandósítás

Mindkét indexet bináris formátumba konvertálják, és speciális kulcsokként tárolják a KvStore-ban.

| Index típus | Mágikus fejléc | Kulcs a KvStore-ban | Sorozatosító függvény |
| --- | --- | --- | --- |
| BM25 | 0x42_4D_32_35 | __agdb_bm25_state | Bm25Index.serialize |
| Vektor | 0x56_45_43_30 | __agdb_vector_state | VectorIndex.serialize |

A Database.flushState függvény felel az indexek sorosításának összehangolásáért és azoknak a tartós kulcs-érték tárolóba való mentéséért.

## Memóriakezelés és szemétgyűjtés

Az AGDB egy többszintű memóriakezelési stratégiát valósít meg, amelyet nagy teljesítményű, állandó tárolásra terveztek. Ötvözi a determinisztikus referenciaszámlálást az azonnali memóriavisszanyerés érdekében, valamint a háttérben futó szemétgyűjtést a ciklusfelismerés és a halomtömörítés céljából. A rendszer x86-64 hardverekre van optimalizálva, JIT-fordítású allokációs szabályokat és zármentes párhuzamos feldolgozási primitíveket alkalmazva.

### RefCountGC: Hivatkozásszámlálás WAL-integrációval

Az AGDB-ben a főmemória életciklusát a RefCountGC kezeli. Ez a komponens a Write-Ahead Log (WAL) integrálásával olyan hivatkozásszámlálást biztosít, amely szálbiztos és összeomlásbiztos is egyben.

### A referenciaszámlálás életciklusa

1.   Hivatkozásszám növelése/csökkentése: Amikor egy PersistentPtr-re hivatkoznak, vagy az elhagyja a hatókört, az incrementRefCount vagy a decrementRefCount metódus hívódik meg.
2.   WAL-alapú frissítések: Az ACID-kompatibilitás biztosítása érdekében a hivatkozásszám-változásokat .ref_count_inc vagy .ref_count_dec rekordként rögzítik a WAL-ban, mielőtt a PersistentHeap-ben található ObjectHeader frissülne
3.   MPSC-kötegelés: A zárversengés és a WAL-terhelés csökkentése érdekében az AGDB egy több-termelő, egy-fogyasztó (MPSC) sorbaállítást (rc_batch) használ a referenciaszámlálási műveletek kötegelésére. Ezeket a flushBatchedRefCounts parancs segítségével rendszeresen kiürítik.
4.   Halasztott szabadítási verem: A hivatkozásszámuk nulla objektumokat a deferred_frees nevű LockFreeStack veremre helyezzük. Ezzel elkerülhetők a mély rekurzív szabadítások során fellépő hosszú leállások, mivel a rendszer a drainDeferredFrees segítségével fokozatosan ürítheti a veremet.

### Ciklusmegszakító mechanizmus

Mivel a hagyományos referenciaszámlálás nem képes felszabadítani a ciklikus gráfokat, az AGDB egy cycle_breakers listát tart fenn. Ha a vizsgálatok során potenciális ciklusokat észlelnek, a háttérben futó gyűjtő nyomon követi és feldolgozza ezeket a mutatókat, hogy megszakítsa a körkörös hivatkozásokat és felszabadítsa a szivárgó memóriát.

### PartitionedGC és Mark-and-Sweep

A ciklusok visszanyerése és a halomfragmentáció optimalizálása érdekében az AGDB párhuzamos PartitionedGC-t alkalmaz. Ez a gyűjtő a halomszegmensek (partíciók) között működik, hogy a rendszer teljes leállását okozó szüneteket minimálisra csökkentsék.

### Párhuzamos jelölés-és-törlés

A GC egy GCContext objektumot használ a gyűjtési ciklus állapotának nyomon követésére

- Jelölési fázis: A gyűjtő a gyökérből (pl. a ResidentObjectTable-ből) kiindulva végigjárja az objektumgráfot. A WorkStealDeque-t használja (amelyet a worklist segítségével valósítanak meg) a vizsgálati feladatok több szálra történő elosztásához.
- Tisztítási fázis: Az object_registry-ben található, a jelölési fázis során el nem ért objektumokat véglegesítik és felszabadítják.

### Adatáramlás: GC-művelet

A GCOperationDescriptor meghatározza azokat a csomagokat, amelyeket a szálak közötti GC-feladatok kommunikációjához használnak, biztosítva, hogy az OP_FREE vagy az OP_MARK műveletek integritás-ellenőrzéssel (CRC32c) kerüljenek végrehajtásra

### Adaptív és aréna-elosztási minták

Az AGDB különféle teljesítményigényekhez speciális allokátorokat használ:

1.   JIT AdaptiveAllocator: A gyakori memóriakiosztásokhoz az AGDB egy JIT-fordítású osztályozót használ. Ez nyomon követi az allokációs statisztikákat (AllocStats)  és futásidőben optimalizált x86-64 gépi kódot generál, hogy az allokációs méreteket osztályokba sorolja (tiny, small, medium, large, huge)
2.   ArenaAllocator: Rövid élettartamú, kéréshez kötött adatokhoz használatos. Nagy, összefüggő puffereket allokál, és a mutató (pos) egyszerű továbbításával gyors, állandó időbeli allokációt biztosít.
3.   Virtuális tömörítés: A PersistentAllocator a GC-vel együttműködve virtuális tömörítést hajt végre: az objektumokat áthelyezi a töredezettség csökkentése érdekében, miközben frissíti a RelativePtr eltolásait, hogy a mutatók érvényessége a tartós térképen belül megmaradjon.

### Műszaki architektúra-ábrák

### A memóriakezelő komponensek közötti kapcsolatok

Ez az ábra összeköti a természetes nyelvű fogalmakat a Zig-specifikus struktúrákkal és fájlokkal.

### Hivatkozásszámlálás és a WAL-adatáramlás

Ez az ábra bemutatja, hogyan kerül rögzítésre a hivatkozásszám változása, és hogyan kerül végül felszabadításra.

### A megvalósítás részletei: GC-metadatok

A GCObjectInfo struktúra a halomban található minden kezelt objektum elsődleges metaadat-rekordja.

| Mező | Típus | Leírás |
| --- | --- | --- |
| ref_count | u32 | Az objektumra mutató hivatkozások jelenlegi száma. |
| jelzők | u32 | Állapotjelzők (pl. megjelölve, rögzítve, véglegesítve). |
| schema_id | u32 | Az objektum típusának/sémájának azonosítója. |
| first_ref_offset | u32 | Az első belső PersistentPtr eltolása a beolvasáshoz. |
| scan_fn_offset | u64 | A JIT-lefordított vagy statikus szkennelő függvény eltolása. |

### Memóriastatisztikák és felügyelet

A rendszer a GCStats struktúrán keresztül követi nyomon a GC teljesítményét, amely olyan mutatókat tartalmaz, mint a cycles_broken, a bytes_freed és a total_time_ns. Ezeket a statisztikákat a JITAllocPolicy használja fel az allokációs küszöbértékek dinamikus beállításához az alkalmazás „leggyakrabban használt útvonalai” alapján

## Futtatási környezet és szerver

A Runtime és a Server komponensek alkotják az AGDB végrehajtási és hozzáférési rétegeit. A Runtime központi koordinátorként működik: inicializálja és összekapcsolja az összes alapvető motor-alrendszert (tárolás, tranzakciók, biztonság és hardveres gyorsítás). A Server komponens nagy teljesítményű HTTP-felületet biztosít, amely hálózati hozzáférést tesz lehetővé az alapul szolgáló adatbázis-motorhoz.

### Futtatási időbeli koordináció

A Runtime struktúra az adatbázis-példány életciklusának fő tárolója. Ez kezeli az összes főbb alrendszer függőségbeépítését és inicializálási sorrendjét. A Runtime.init metódus meghívásakor a rendszer egy sor kritikus beállítási feladatot hajt végre:

1.   Biztonság és hardver: Inicializálja a SecurityManager-t, és felismeri a rendszer NumaTopology-ját
2.   Tároló-leképezés: Leképezi a PersistentHeap-et, és – amennyiben be van állítva – hozzárendeli azt a NUMA-csomópontokhoz
3.   Hiba utáni helyreállítás: A RecoveryEngine.recover() műveletet a Write-Ahead Log (WAL) segítségével hajtja végre, még mielőtt bármilyen allokációra sor kerülne.
4.   Tranzakciós összekapcsolás: Beállítja a TransactionManager-t, és bekapcsolja a PersistentAllocator-t a tranzakciós visszavonás/ismétlés támogatásához
5.   Számítási regisztráció: Ha engedélyezve van, a GPUContext segítségével regisztrálja a cosine_similarity-hez hasonló GPU-kerneleket

Az inicializálási sorrend és a tranzakciós életciklus részletes leírását lásd

### A futásidejű komponensek vázlatos ábrája

Ez az ábra a magas szintű futásidejű alrendszereket a hozzájuk tartozó konkrét kódelemekhez rendeli.

### Szerverkomponens

A szerver HTTP-n keresztül teszi elérhetővé az AGDB-t, így a beágyazott motort hálózati szolgáltatássá alakítja. A párhuzamos kapcsolatok kezeléséhez egy nem blokkoló std.net.Server-t használ, és a kéréseket továbbítja az adatbázis-példánynak.

### API-funkciók

A szerver RESTful felületet biztosít a szokásos adatbázis-műveletekhez:

- Egészség és statisztikák: /health, /version és /stats
- Adatkezelés: POST /records beillesztéshez, /records/{id} lekéréshez és DELETE /records/{id}
- Keresés: POST /search hibrid vagy vektoros lekérdezések végrehajtásához
- Karbantartás: A /compact és /flush parancsok a tárhelyoptimalizálás elindításához

### A szerver futási folyamata

Ez az ábra egy HTTP-kérés útját követi nyomon a hálózattól a szerver logikáján át az adatbázis-motorig.

### CLI felület

A projekt tartalmaz egy parancssori eszközt a helyi interakcióhoz és a rendszergazdai feladatok elvégzéséhez. A parancssori felület a felhasználói parancsokat közvetlenül az adatbázis-műveletekhez rendeli hozzá, megkerülve a hálózati réteget, így alacsonyabb késleltetést és egyszerűbb helyi kezelést biztosítva. A konfigurációt a RuntimeConfig segítségével kezeli, amelynek segítségével megadhatók a konkrét adatkönyvtárak és a WAL-útvonalak.

A rendelkezésre álló parancsokról és a fordítási beállításokról szóló dokumentációt lásd

### Beállítások és statisztikák

Mind a Runtime, mind a Server kifejezetten erre a célra szolgáló struktúrák segítségével széles körben konfigurálható:

| Konfigurációs struktúra | Főbb mezők | Cél |
| --- | --- | --- |
| RuntimeConfig | heap_size, wal_path, enable_encryption, gc_threshold | A motor alapvető viselkedését és az erőforrás-korlátokat szabályozza.  |
| ServerConfig | cím, port, api_token, max_body_size | A hálózati figyelő és a biztonsági fejlécek beállítása.  |

A Runtime a RuntimeStats segítségével valós idejű mutatókat is nyomon követ, többek között a heap-használatot, a tranzakciók számát és a szemétgyűjtés teljesítményét.

## Futtatási koordináció

A Runtime az AGDB központi koordinációs motorjaként működik, és az összes alapvető alrendszer életciklusáért és integrációjáért felel. Feladata a nyers lemezfájlokból egy teljesen működőképes, tranzakciós adatbázis-környezetbe való átmenet kezelése.

### Inicializálási sorrend

A Runtime.init függvény egy szigorú lépéssorozatot hajt végre az adatok integritásának biztosítása és a hardver optimalizálása érdekében, mielőtt bármilyen művelet végrehajtására sor kerülne.

### 1. Biztonság és hálózati felépítés

- SecurityManager: Elsőként inicializálódik a PersistentHeap és a WAL esetleges visszafejtésének kezelése érdekében. Kezelése alá tartoznak a főkulcsok, valamint az AES-GCM/ChaCha20-Poly1305 állapot
- NUMA-felismerés: Ha az enable_numa beállítás szerepel a RuntimeConfig fájlban, a rendszer felismeri a CPU-topológiát

### 2. Tárolási leképezés

- PersistentHeap: A fő adatfájl memóriába leképezett. Ha a NUMA engedélyezve van, a memóriatartományt a numa_mod.bindMemoryToNode függvény segítségével kifejezetten a helyi csomóponthoz rendelik, hogy minimalizálják a processzorok közötti késleltetést
- Előreírási napló (WAL): A biztonsági kontextussal inicializálva, hogy titkosított naplóbejegyzések létrehozását tegye lehetővé

### 3. Helyreállítás és vezetékezés

- RecoveryEngine: Mielőtt bármilyen új tranzakció elindulna, a RecoveryEngine átvizsgálja a WAL-t és a heapet a Redo/Undo fázisok végrehajtása érdekében, így biztosítva, hogy a rendszer összeomlás után is konzisztens állapotban legyen
- PersistentAllocator: A halom szabad listáinak és szelvényosztályainak kezelésére szolgál
- TransactionManager: Kapcsolatban áll mind a WAL-lal, mind az Allocatorral. Kritikus lépés az undoAllocationThunk regisztrálása, amely lehetővé teszi a TransactionManager számára, hogy automatikusan visszavonja a heap-allokációkat, ha egy tranzakció sikertelen lesz

### 4. Gyorsítás és nyomkövetés

- GPU-regisztráció: Ha engedélyezve van, a GPUContext betölti a külső könyvtárakat, és regisztrálja a kerneleit, például a cosine_similarity-t a vektoros kereséshez
- walAppendHook: Ha a nyomkövetés engedélyezve van, akkor a WAL-hoz egy hook kapcsolódik. A naplóhoz hozzáfűzött minden rekordot a TraceWriter egyidejűleg egy .replay fájlba továbbít.

### A futásidejű inicializálás folyamata

## Runtime.init vezérlési folyamat

Források:   175

### Beállítási lehetőségek

A RuntimeConfig struktúra határozza meg a motor működési paramétereit.

| Opció | Típus | Leírás |
| --- | --- | --- |
| heap_size | u64 | A memóriába leképezett halom teljes mérete  |
| enable_encryption | bool | Az AES/ChaCha20 titkosítás be- és kikapcsolása a heap és a WAL esetében  |
| gc_threshold | u64 | A RefCountGC elindítását megelőző műveletek száma |
| enable_numa | bool | Engedélyezi a topológiát figyelembe vevő memóriakötést  |
| enable_jit_alloc | bool | Az AdaptiveAllocator aktiválása JIT-feladatokhoz  |
| enable_trace | bool | Engedélyezi a walAppendHook futásvisszajátszását  |

### A tranzakciós allokáció életciklusa

A Runtime egy magas szintű API-t biztosít a memóriakezeléshez, amely integrálva van a TransactionManagerrel. Ez garantálja, hogy még a nyers memóriakiosztások is összeomlásbiztosak és atomikusak legyenek.

### Allokálás és felszabadítás

- allocate(size, alignment): Meghívja a PersistentAllocator objektumot. Ha tranzakció van folyamatban, a memóriakiosztást nyomon követik
- free(ptr): A felszabadítási műveletet rögzíti a WAL-ban. A tényleges memória csak a tranzakció véglegesítése után kerül vissza a felszabadítási listába

### A Visszavonás-gondolat

Az undoAllocationThunk egy visszahívási függvény, amelyet a TransactionManager használ a visszavonás során. Ha egy memóriát allokáló tranzakció meghiúsul, ezt a thunkot hívják meg, hogy a memóriát azonnal visszaadja a PersistentAllocator-nak, így megakadályozva a tartós halomban fellépő memóriaszivárgást.

### Entitás-leképezés: Memóriakezelés

## Tranzakciós allokációs leképezés

### Pillanatképek és a GC-kezelés

### Pillanatfelvétel készítése

A futásidejű környezet rendelkezésre bocsátja a createSnapshot() metódust, amely a SnapshotManager-hez delegál. Ez a folyamat a PersistentHeap egy adott időpontban készült másolatát hozza létre anélkül, hogy blokkolná az aktív tranzakciókat, és a PersistentHeap.flush() mechanizmust használja annak biztosítására, hogy a lemezen található adatok naprakészek legyenek

### Memória-felszabadítás

A RefCountGC a Runtime.collectGarbage() metóduson keresztül hívódik meg.

1.   Feldolgozza az MPSC (Multi-Producer Single-Consumer) szabad sorát
2.   Azokat az objektumokat azonosítja, amelyek ObjectHeader-ében a ref_count értéke nulla.
3.   Komplex gráfstruktúrák esetén a „Cycle Breaker” algoritmust hajtja végre.
4.   Egy külön GC-tranzakción belül ingyenes parancsokat ad ki a PersistentAllocator segítségével

### Rendszerstatisztikák

A Runtime.getStats() függvény a rendszer egészéből gyűjt adatokat, és egységes áttekintést nyújt a következőkről:

- A memóriaterület kihasználtsága (használt vs. teljes).
- Tranzakciós átviteli sebesség és a WAL mérete.
- A GC hatékonysága (összegyűjtött objektumok száma a beolvasottakhoz viszonyítva).

## CLI felület

Az agdb parancssori felület közvetlen hozzáférést biztosít az AGDB adatbázis-motor adminisztrációs és üzemeltetési funkcióihoz. Lehetővé teszi a felhasználók számára az adatbázisok inicializálását, a rekordokon végzett CRUD-műveleteket, a hibrid keresések (BM25 és Vector) végrehajtását, valamint az adatbázis-karbantartási feladatok – például a tömörítés és a pillanatfelvételek készítése – kezelését.

A CLI elsősorban a src/cli.zig fájlban van megvalósítva, és a src/cli_main.zig fájlban található belépési ponton keresztül hívják meg.

### Építészet és belépési pont

A CLI egy szabványos parancsszerkezet-alapú architektúrát követ, amelyben a src/cli_main.zig fájlban található fő függvény fogadja a folyamat argumentumait, majd azokat továbbítja az agdb.cli.run függvénynek.

### A végrehajtás menete

1.   Argumentumok feldolgozása: a cli_main.zig a std.process.argsAlloc függvényt használja a nyers argumentumok lekérésére
2.   Átadás: Az argumentumokat (a bináris név kivételével) átadjuk a cli.run függvénynek
3.   Opciók feldolgozása: a cli.run végigfut az argumentumokon, hogy kitöltse a CliOptions struktúrát, amely olyan globális beállításokat tartalmaz, mint az adatkönyvtár és a beágyazási dimenziók
4.   Parancs továbbítása: Az első pozíciós argumentum alapján a CLI a parancsot a megfelelő parancskezelőkhöz továbbítja (pl. cmdPut, cmdSearch)

### Parancs-küldés leképezése

| Parancs | Kezelőfüggvény | Adatbázis-művelet |
| --- | --- | --- |
| init | cmdInit | Database.open + flush |
| put | cmdPut | Database.putBytes |
| get | cmdGet | Database.get |
| del | cmdDel | Database.delete |
| keresés | cmdSearch | Database.searchHybrid |
| vektoros keresés | cmdVectorSearch | Database.searchVector |
| compact | cmdCompact | Database.compact |

### Beállítás és inicializálás

A CLI a CliOptions segítségével konfigurálja az adatbázis-példányt. Ezeket az opciókat a parancsnév előtt megadott globális jelzőkkel lehet beállítani.

### Világzászlók

- --data / -d: Beállítja a data_dir mappát (alapértelmezett: ./agdb-data)
- --dim: Beállítja az embedding_dim értéket a vektoros műveletekhez (alapértelmezett: 256)
- --bind / --port: A serve parancs hálózati paramétereinek beállítása

Az openDb segédfüggvényt szinte minden parancskezelő használja az adatbázis-motor ezen paraméterekkel történő inicializálásához

A CLI és az adatbázis közötti inicializálási folyamat

### Adatkezelés

### Beillesztési műveletek

A CLI háromféle adatbeolvasási módot támogat:

1.   put: Karakterláncot fogad a parancssorból
2.   put-file: Beolvassa egy fájl tartalmát a lemezről, és rekordként tárolja
3.   put-json: JSON-fájl vagy karakterlánc feldolgozása strukturált adatok beviteléhez

Minden „put” művelet végső soron a db.putBytes(kind, explicit_id, body, tags) metódust hívja meg. Ez a metódus kezeli a rekord létrehozását, frissíti a szöveges BM25-indexet, és szükség esetén generálja a beágyazásokat.

### Keresési műveletek

A CLI a cmdSearch parancs segítségével teszi elérhetővé az AGDB hibrid keresési funkcióit. Ez a parancs a Database.searchHybrid metódushoz kapcsolódik, amely ötvözi a kulcsszavak relevanciáját (BM25) és a szemantikai hasonlóságot (Vector).

A keresés végrehajtásának folyamata

Források: 138

### Karbantartás és közművek

A CLI parancsokat biztosít az adatbázis életciklusának és állapotának kezeléséhez:

- init: Kényszeríti az adatbázis-könyvtár létrehozását, és frissíti a kezdeti állapotot
- stats: Az adatbázisra vonatkozó metaadatokat ad vissza, például a rekordok számát és az indexek méretét
- compact: Elindítja a Database.compact() rutint, amely visszanyeri a törölt rekordok által elfoglalt helyet, és optimalizálja a tárolást
- flush: Gondoskodik arról, hogy az összes memóriában lévő WAL (Write-Ahead Log) bejegyzés és a módosított oldalak lemezre kerüljenek
- serve: Elindítja a server.zig fájlban meghatározott HTTP-kiszolgálót, lehetővé téve a CLI-program számára, hogy önálló adatbázis-kiszolgálóként működjön

## Felhőréteg: többfelhasználós szolgáltatás

Az agdb-cloud szolgáltatás többfelhasználós, biztonságos és elszigetelt környezetet biztosít az AGDB-példányok tárolásához. Egyrészt irányító rétegként működik a felhasználói fiókok életciklusának kezeléséhez, másrészt adatkezelő rétegként a nagy teljesítményű adatbázis-lekérdezések elszigetelt tesztkörnyezeti folyamatokhoz történő továbbításához.

### A rendszer áttekintése

A felhőréteg három fő összetevőből áll, amelyeket a fő belépési ponton inicializálnak:

1.   Nyilvántartás: A bérlői rekordok és a hitelesítés állandó metadatatára
2.   Folyamatok táblázata: egy futásidejű nyilvántartás, amely nyomon követi az aktív homokdoboz-folyamatokat, és elősegíti a folyamatok közötti kommunikációt (IPC)
3.   Felhőszerver: egy HTTP/1.1-kiszolgáló, amely a REST API-t teszi elérhetővé, és összehangolja a nyilvántartó és a folyamatlista működését

### Általános felépítés

Az alábbi ábra bemutatja a kapcsolatot a HTTP-elülső felület, az irányítási nyilvántartás és az elszigetelt bérlői futtatási környezetek között.

Felhőszolgáltatási komponensek közötti kapcsolatok

### HTTP API és felhasználói felület

A CloudServer kezeli a bejövő TCP-kapcsolatokat, és az URL-útvonal alapján továbbítja azokat a megfelelő feldolgozókhoz. A következőket biztosítja:

- Végpontok kezelése: /v1/auth/register, /v1/auth/login és API-kulcsok kezelése.
- Adatbázis-végpontok: /v1/query, /v1/search és /v1/insert, amelyek a bérlői homokozókba irányulnak.
- Beágyazott felhasználói felület: Beépített HTML-felületet és dokumentációs oldalt biztosít

További részletekért lásd

### Bérlői nyilvántartás és biztonság

A nyilvántartó kezeli a TenantRecord struktúrákat, amelyek olyan fontos metaadatokat tárolnak, mint a tenant_id, a hash-elt hitelesítő adatok és a fájlrendszer data_path eleme

- Elszigetelés: Minden bérlőnek egy egyedi könyvtár van hozzárendelve az AGDB_DATA_ROOT alatt
- Hitelesítés: Az API-kulcsokhoz Blake3 hash-algoritmust használ, és a gyors keresés érdekében névtereket biztosít (pl. email:, apikey:, tenant:)

További részletekért lásd

### Sandbox-elkülönítés

A biztonság és az erőforrások méltányos elosztásának biztosítása érdekében az agdb-cloud nem futtatja a bérlői kódot a fő folyamatban. Ehelyett a spawnTenantSandbox parancsot használja, hogy Linux névterek és cgroups v2 segítségével elszigetelt környezetet hozzon létre.

- Erőforrás-korlátozások: Korlátozza a bérlőnkénti memóriát, CPU-kapacitást és PID-számot.
- Sandbox Runner: Egy különálló bináris fájl (sandbox_runner), amely visszavonja a jogosultságokat, és inicializálja az agdb-runtime-ot az adott bérlő számára.

További részletekért lásd

### Kérelmek továbbítása és IPC

A router feladata, hogy a HTTP-kérelmet a megfelelő SandboxHandle-hez továbbítsa.

- Folyamatkezelés: Ha egy bérlői sandbox nem fut, az útválasztó a ProcessTable-en keresztül elindít egy új példányt
- IPC-protokoll: A kommunikáció egyedi bináris protokollon keresztül, Unix-domén-socketeken keresztül történik
- Állapotmentes üzemmód: Támogatja az „állapotmentes” végrehajtási üzemmódot, amelyben a WAL (Write-Ahead Log) bejegyzéseket egy távoli végponthoz továbbítják

További részletekért lásd

### Kód-entitás-térkép

Ez az ábra a logikai felhőműveleteket azokhoz a konkrét Zig-struktúrákhoz és függvényekhez rendeli, amelyek azokat megvalósítják.

Logikai műveletek és kódentitások közötti leképezés

Források: 210 45 108 120

## HTTP API-útmutató (v1)

Az AGDB Cloud API (v1) egy RESTful interfész, amelyet az agdb-cloud biztosít a többfelhasználós adatbázis-terhelések kezeléséhez. Ez szolgál irányító rétegként a felhasználók regisztrációjához, a fiókkezeléshez és az API-kulcsok életciklusához, miközben az adatbázis-műveletekhez nagy teljesítményű proxyként működik az adatréteg (felhasználói sandboxok) felé.

### Az architektúra áttekintése

Az API-t a CloudServer struktúra biztosítja, amely egy egyedi HTTP-megvalósítást integrál a Registry-vel (a bérlői metaadatok állandósítása) és a Router-rel (a kérések elosztása az elszigetelt tesztkörnyezetekbe).

### Életciklus-diagram lekérése

Ez az ábra bemutatja a beérkező HTTP-kérés útját a bérlőnként elszigetelt tesztkörnyezetben történő végrehajtásig.

## Kérés átirányítása a tesztkörnyezetbe

### Hitelesítés és fejlécek

Minden nem nyilvános végponthoz Bearer token szükséges az Authorization fejlécben.

- Fejléc: Engedélyezés: Bearer agdb_<hex_string>
- CORS: A szerver megvalósítja a sendCors függvényt, amely a különböző eredetű kérésekhez szükséges szabványos fejléceket biztosítja, beleértve az Access-Control-Allow-Origin:  és az Access-Control-Allow-Methods: GET, POST, DELETE, OPTIONS fejléceket.
- Tartalomtípus: A kérések és válaszok az application/json formátumot használják.

### Végpontok áttekintése

### 1. Egészség és felfedezés

| Végpont | Módszer | Leírás |
| --- | --- | --- |
| /v1/health | GET | Visszaadja: {"status":"ok","version":"2.4.0"} |
| /docs | GET | A beágyazott HTML-dokumentációt szolgáltatja  |
| / | GET | Az AGDB Console felületét szolgálja ki  |

### 2. Hitelesítés és fiókkezelés

Ezeket a műveleteket a RegistrationHandler kezeli

| Végpont | Módszer | Hitelesítés | Leírás |
| --- | --- | --- | --- |
| /v1/auth/register | POST | Nyilvános | Új bérlőt regisztrál e-mailen keresztül. Visszaadja a tenant_id-t és a kezdeti api_key-t |
| /v1/account | DELETE | Bearer | Visszavonja a bérlőt, leállítja a sandboxot, és rekurzív módon törli az összes adatot a lemezről  |

### 3. API-kulcsok kezelése

Az ApiKey-struktúrák és a Blake3 hash-algoritmus segítségével kezelhető

| Végpont | Módszer | Leírás |
| --- | --- | --- |
| /v1/apikeys | GET | Az azonosított bérlő összes aktív API-kulcsának felsorolása. |
| /v1/apikeys | POST | Új, „agdb_” előtaggal rendelkező kulcsot hoz létre, és annak hash-értékét a nyilvántartásba menti  |
| /v1/apikeys/rotate | POST | Az elsődleges kulcsot cseréli, és a régit visszavonja. |

### 4. A homokozó életciklusa

A homokozók Linux névterek és cgroupok segítségével kezelt, elszigetelt folyamatok

| Végpont | Módszer | Leírás |
| --- | --- | --- |
| /v1/sandbox/status | GET | Visszaadja a bérlő sandboxjának állapotát (Fut/Leállítva) és erőforrás-használatát. |
| /v1/sandbox/restart | POST | Erőszakosan leállítja az aktuális sandbox_runner-t, és elindít egy újat. |

### 5. Adatbázis-műveletek (adatfelület)

Ezeket a kéréseket Unix-domén-socketeken (IPC) keresztül továbbítják a sandbox_runnernek.

| Végpont | Módszer | Adatcsomag | Leírás |
| --- | --- | --- | --- |
| /v1/query | POST | JSON-lekérdezés | Nyers AGDB-lekérdezést hajt végre. |
| /v1/search | POST | Keresési paraméterek | BM25, vektoros vagy hibrid keresést hajt végre  |
| /v1/insert | POST | Rekord(ok) | Objektumok tranzakciós beillesztése. |
| /v1/stats | GET | N/A | Az adatbázis-motor statisztikáit adja vissza (heap méret, WAL állapot, tranzakciók száma). |

### Adatszerkezetek és IPC-protokoll

Az API-réteg a következő alapvető entitásokkal áll kapcsolatban:

### Kód-entitás leképezés

## Felhőrétegek közötti entitáskapcsolatok

### IPC üzenetformátum

Amikor az útválasztó egy kérést továbbít egy sandboxba, egy Unix-socket segítségével egy egyszerűsített bináris protokollt használ:

1.   Opkód: 1 bájt (pl. 0x01 lekérdezéshez, 0x02 kereséshez).
2.   Hossz: 4 bájt (a hasznos adat Little-endian formátumú mérete).
3.   Adattartalom: a kérés törzsének JSON-karaktersorozata.

Ezt követően az útválasztó egy szemaforon (a SandboxHandle része) várja, hogy a homokozó jelezze a befejezést

- src/api.zig:  (Handle megvalósítás)
- src/cloud/http_server.zig:  (CloudServer és a kérések továbbítása)
- src/cloud/registration.zig:  (Fiók/Regisztrációs logika)
- src/cloud/apikey.zig:  (Kulcsgenerálás és hash-képzés)
- src/cloud/router.zig:  (IPC és homokozó-kezelés)
- src/cloud/docs.html:  (API-dokumentáció tartalma)

## Bérlői nyilvántartás és API-kulcskezelés

A Registry alrendszer az AGDB felhőréteg vezérlőrétege. Ez kezeli a bérlők életciklusát, biztosítja a fájlrendszer elszigeteltségét, valamint API-kulcsok segítségével biztosítja a biztonságos hitelesítést. Az AGDB adatbázis-motor egy belső példányát használja a bérlői metaadatok tárolására, valamint az e-mail címek, a bérlői azonosítók és a hash-elt API-kulcsok közötti leképezés fenntartására.

### Bérlői metaadatok és rekordstruktúra

A rendszerben minden bérlőt egy TenantRecord objektum képvisel. Ez a struktúra bináris blobként van tárolva a rendszerleíró adatbázis kulcs-érték tárolójában, és egy egyedi tenant_id azonosító alapján van indexelve.

| Mező | Típus | Leírás |
| --- | --- | --- |
| tenant_id | u64 | A bérlő számára véletlenszerűen kiosztott egyedi azonosító. |
| email_hash | [32]u8 | A normalizált e-mail-cím Blake3-hash-értéke. |
| api_key_hash | [32]u8 | Az aktív API-kulcs Blake3-hashja. |
| created_at_unix | i64 | A regisztráció Unix-időbélyege. |
| aktív | u8 | Boole-érték (1: aktív, 0: visszavont). |
| stateless | u8 | Jelző, amely megadja, hogy a bérlő állapotmentes módban működik-e (WAL-végpontok használatával). |
| data_path | [256]u8 | A bérlő elkülönített adatkönyvtárának abszolút elérési útja. |

### Főbb névterek és belső tárhely

A nyilvántartó speciális karakterlánc-előtagokat használ az adatok belső KvStore-on belüli rendszerezéséhez. Ez lehetővé teszi a hatékony keresést különböző dimenziók (e-mail, azonosító vagy kulcs) alapján.

- e-mail:<hex_hash>: Egy hash-elt e-mail-címet társít egy 64 bites bérlői azonosítóhoz. Az ismételt regisztrációk megakadályozására szolgál.
- bérlő:<id>: Összekapcsolja a bérlő azonosítóját a teljes TenantRecord struktúrával.
- apikey:<hex_hash>: A hitelesítéshez egy hash-elt API-kulcsot rendel egy u64 típusú bérlői azonosítóhoz.
- tenant_list: Az összes regisztrált bérlő azonosítójának vesszővel elválasztott listája.

### Adatáramlás: regisztrációtól a kódegységekig

Az alábbi ábra bemutatja a regisztrációs kérelem logikai folyamatát, valamint a folyamatban részt vevő konkrét függvényeket és struktúrákat.

A regisztráció folyamatábrája

### API-kulcsok kezelése

Az API-kulcsok generálásakor egy szabványos előtagot (agdb_) használnak, amelyet hexadecimális kódolású véletlenszerű bájtok követnek. Biztonsági okokból a kulcsokat soha nem tárolják egyszerű szöveg formátumban; csak a Blake3-hash-értéküket mentik el.

### Generálás és hash-funkciók

- Generálás: A generateApiKey függvény egy 64 bájtos, nullával végződő karakterláncot állít elő. A std.crypto.random modul segítségével 29 bájt entrópiát generál, amelyet ezután hexadecimális kódolással kódol.
- Hash-funkció: A hashApiKey a Blake3 kriptográfiai hash-függvényt használja. A null-végjelzés kezelésére úgy kerül sor, hogy a hash-elés előtt levágja a végső nullát.
- Ellenőrzés: a verifyApiKey funkció állandó időbeli összehasonlítást hajt végre a kiszámított hash és a tárolt hash között az időalapú támadások megakadályozása érdekében.

### Kulcsok cseréje és visszavonása

Amikor egy új kulcsot a storeApiKeyHash segítségével tárolnak, a Registry frissíti a TenantRecord-ot, és kezeli az apikey: névteret. Ha korábban már létezett ilyen kulcs, annak hash-bejegyzését törlik az adatbázisból, hogy biztosítsák: azt többé nem lehet hitelesítésre használni.

### Fájlrendszer-elkülönítés és biztonság

A nyilvántartó szigorú elszigeteltséget biztosít azzal, hogy minden bérlőnek egy saját könyvtárat rendel az AGDB_DATA_ROOT alatt.

1.   Útvonal létrehozása: A regisztráció során létrehozásra kerül egy útvonal a {AGDB_DATA_ROOT}/{tenant_id}/ könyvtárban.
2.   Könyvtár létrehozása: A rendszerleíró adatbázis a std.fs.cwd().makePath függvény segítségével hozza létre ezt a könyvtárat.
3.   Tisztítás: A fiók törlésekor (handleDeleteAccount) a rendszer rekurzív törlést hajt végre a bérlő adatkönyvtárában, és eltávolítja a TenantRecord táblaelemet az adatbázisból.

### A rendszerleíró adatbázis szálbiztonsága

A Registry-n végzett összes műveletet (regisztráció, lekérdezés, rotáció) egy std.Thread.Mutex (Registry.mu) védi. Ez biztosítja, hogy a felhőszerverhez érkező párhuzamos HTTP-kérelmek ne okozzanak versenyhelyzeteket a belső kulcs-érték tároló frissítésekor vagy a bérlői azonosítók kiosztásakor.

### API-kulcsos hitelesítési folyamat

Az alábbi ábra bemutatja a hitelesítési folyamatot a HTTP-kérés és a belső nyilvántartás-lekérdezések között.

Hitelesítés és lekérdezési leképezés

Források:   63

## Sandbox-elkülönítés és folyamatkezelés

Az agdb-cloud szolgáltatás többfelhasználós biztonságot és az erőforrások méltányos elosztását biztosítja azáltal, hogy minden felhasználói munkaterhelést egy különálló Linux-szandboxba szeparál. Ez az elszigetelés a cgroup v2 erőforrás-korlátozások, a Linux névterek és a Seccomp rendszerhívás-szűrés kombinációjával valósul meg.

### A homokozó életciklusa és elszigeteltsége

A bérlők elszigetelését a spawnTenantSandbox függvény irányítja. Amikor egy bérlői kérés érkezik, és nincs aktív folyamat, a szerver új környezetet inicializál.

### 1. Erőforrás-korlátozások (cgroup v2)

A folyamat elindítása előtt a rendszer létrehoz egy dedikált cgroup-könyvtárat a /sys/fs/cgroup/agdb/tenant-{id} alatt. A következő kemény korlátokat állítja be:

- Memória: 512 MB-ra korlátozva (memory.max)
- Lapos tárhely: Letiltva (memory.swap.max = 0)
- PID-ek: maximum 64 folyamat/szál
- CPU: A sávszélesség egy mag teljesítményének 50%-ára van korlátozva (500 ms egy másodperces időszakonként)

### 2. Névtér-elkülönítés (clone3)

A sandbox-folyamatot a clone3 rendszerhívás segítségével hozzák létre, kiterjedt névterületi jelzőkkel

- CLONE_NEWPID: A folyamat saját magát 1-es PID-ként azonosítja; a többi bérlői folyamatot nem látja.
- CLONE_NEWNS: Zárt csatlakozási névterület.
- CLONE_NEWNET: Zárt hálózati réteg (csak hurokvisszacsatolás).
- CLONE_NEWIPC: Elszigetelt System V IPC- és POSIX-üzenetsorok.
- CLONE_NEWUTS: Elszigetelt gazdagépnév és domainnév.
- CLONE_NEWUSER: A bérlői folyamatot egy jogosultságokkal nem rendelkező felhasználóhoz rendeli hozzá.

### 3. Megállapodás és végrehajtás

A szinkronizációs cső a szülő- és a gyermekfolyamatok összehangolására szolgál. A szülőfolyamat beállítja az uid_map és a gid_map fájlokat úgy, hogy a belső root felhasználót egy jogosultságok nélküli gazdagép-UID-hez rendelje, mielőtt a csövön keresztül jelzést küldene a gyermekfolyamatnak a folytatásra. A gyermekfolyamat ezután az execve paranccsal futtatja a sandbox_runner bináris fájlt.

### A sandbox generálási adatfolyama

| Lépés | Objektum | Művelet | Fájlhivatkozás |
| --- | --- | --- | --- |
| 1 | agdb-cloud | /sys/fs/cgroup/agdb/tenant-{id} létrehozása |  |
| 2 | agdb-cloud | A memory.max és a cpu.max beállítása |  |
| 3 | clone3 | Gyermekfolyamat létrehozása CLONE_NEWPID-del | CLONE_NEWNS | ... |  |
| 4 | Szülő | Írás /proc/{pid}/uid_map |  |
| 5 | Gyermek | execvesandbox_runner |  |

### Sandbox Runner és Seccomp

A sandbox_runner az elszigetelt névterületen belüli belépési pontként működik. Az agdb futtatókörnyezet elindítása előtt elvégzi a környezet végső biztonsági megerősítését.

### Fájlrendszer-börtön

A futtató a mount és a pivot_root (vagy bind mount) parancsok segítségével létrehoz egy minimális gyökérfájlrendszert:

- Privát tmpfs-ek: Egy 128 MB-os tmpfs csatlakozik a /tmp/agdb-{id} könyvtárba
- Csak olvasható rendszerkönyvtárak: az /usr, /lib és /bin könyvtárak csak olvashatóként vannak csatlakoztatva
- Eszközfehérlista: Csak a /dev/null, a /dev/zero és a /dev/urandom áll rendelkezésre
- Bérlői adatok: A bérlő saját adatkönyvtárát a /data könyvtárba kötötték

### Rendszerhívás-szűrés (Seccomp)

A futtatóprogram a Seccomp segítségével szigorú BPF-szűrőt alkalmaz a rendszermag támadási felületének korlátozására. Csak 81 meghatározott rendszerhívás engedélyezett. Bármely nem engedélyezett rendszerhívás meghívására tett kísérlet a folyamat azonnali leállítását eredményezi a SECCOMP_RET_KILL_PROCESS parancs segítségével.

### Folyamatirányítás

A ProcessTable az összes aktív SandboxHandle-objektum központi nyilvántartása. Szálbiztos, és a bérlői folyamatok életciklusát kezeli.

### A ProcessTable szerkezete

A táblázat az aktív sandboxokat és a függőben lévő IPC-kérelmeket tartja nyilván:

- slots: egy SandboxHandle-t tartalmazó tömb, amelyben a PID, az IPC-fájl leírók és az események időbélyegei találhatók
- epoll_fd: A runDispatchLoop használja az IPC-soketek beérkező válaszainak figyelésére
- pending_requests: A kérelemazonosítók és a WaitingRequest struktúrák közötti leképezés, amelyek szemaforokat használnak a szálak szinkronizálásához

### A Sandbox megszüntetése

A homokozók három esetben kerülnek megsemmisítésre:

1.   Inaktivitási időkorlát: a reapIdleSandboxes olyan kezelőket keres, amelyeknél több mint 10 perce nem történt tevékenység
2.   Zombiprogramok eltávolítása: a reapZombies a wait4 parancsot használja az összeomlott vagy lezárt folyamatok felismerésére
3.   Kifejezett visszavonás: Amikor egy bérlőt a TenantLifecycle.destroyTenant metódussal törölnek

A leállításhoz a destroySandbox függvényt kell meghívni, amely a cgroup.kill parancs segítségével leállítja a cgroupban található összes folyamatot

### Folyamat-táblák közötti interakció

## A Sandbox regisztrációja és a kérések szinkronizálása

### Állapotmentes és állapotfüggő üzemmódok

A sandbox viselkedése a TenantRecord állapotmentes jelzőjéhez igazodik

- Állapotfüggő (alapértelmezett): Az adatbázis a homokozó /data csatlakozási pontján belül található helyi állandó tárolón működik.
- Állapotmentes: A sandbox_runner inicializálása az AGDB_WAL_ENDPOINT környezeti változóval történik. Ebben a módban a futtatókörnyezet nem a helyi lemezt használja a Write-Ahead Log tárolására; ehelyett a wal_transport modul segítségével a WAL-rekordokat a megadott végponthoz továbbítja.

### A Sandbox metaadatok entitás-térképe

## Kódelemek a tesztkörnyezet kezeléséhez

## Kérelmek továbbítása és IPC

A Router az agdb-cloud szolgáltatás központi irányítója. Ez koordinálja a kérések életciklusát a kezdeti HTTP-beolvasástól és hitelesítéstől egészen a bérlői sandboxon belüli biztonságos végrehajtásig. A felhő vezérlőrétege és az adatréteg (sandboxok) közötti kommunikáció egy egyedi folyamatok közötti kommunikációs (IPC) protokollon keresztül történik, Unix domain socketeken keresztül.

### 4.4.1 A kérések feldolgozásának folyamata

A Router.handleHttpRequest függvény valósítja meg az útválasztás alapvető logikáját. Szigorúan meghatározott sorrendet követ, hogy biztosítsa a bérlők elszigeteltségét és az erőforrások rendelkezésre állását.

1.   Hitelesítés: Ellenőrzi az „Authorization: Bearer <key>” fejléce érvényességét
2.   Rendszerleíró adatbázis-lekérdezés: lekérdezi a rendszerleíró adatbázist, hogy az API-kulcsot egy TenantRecord-hoz rendelje
3.   Folyamatlista-ellenőrzés: Ellenőrzi, hogy a tenant_id azonosítóhoz tartozó sandbox már fut-e
4.   Kétszer ellenőrzött zárolásos létrehozás: Ha nincs létező sandbox, akkor megszerez egy spawn_mutex-et, és újra ellenőrzi a táblázatot, mielőtt a sandbox.spawnTenantSandbox metódust meghívná
5.   IPC-küldés: A kérés tartalmát az ipc.sendMessage függvény segítségével elküldi a sandboxnak a sandbox saját ipc_fd-jének használatával
6.   Szinkronizálás: A hívó szál egy WaitingRequest szerkezeten belüli szemaforon vár
7.   Válasz lekérése: Amint a szemafor jelzést kap (vagy letelik a 30 másodperces időkorlát), a válasz a megosztott pufferből visszakerül az HTTP-válaszba

### Küldési sorrend kérése

Az alábbi ábra bemutatja a beérkező HTTP-kérés és a homokozóban történő végrehajtás közötti folyamatot.

### 4.4.2 IPC-protokoll

Az IPC-mechanizmus egy bináris protokollt használ, amelyet alacsony késleltetésű kommunikációra terveztek Unix-socketpárok felett. Minden üzenet egy rögzített méretű fejlécből áll, amelyet egy opcionális, változó hosszúságú hasznos adat követ.

### Protokoll leírás

- Mágikus szám: 0x47444241 (GDBA kis-endian formátumban)
- Fejléc mérete: 20 bájt
- Az opkód (msg_type): 0x01 a szokásos adatbázis-lekérdezésekhez használatos

| Eltolás | Típus | Mező | Leírás |
| --- | --- | --- | --- |
| 0 | u32 | magic | IPC_MAGIC értéknek kell lennie |
| 4 | u64 | request_id | Monoton azonosító a kérés/válasz pároshoz |
| 12 | u8 | msg_type | Művelettípus (pl. 0x01 lekérdezés esetén) |
| 13 | u8 | állapot | 0: sikeres, nullától eltérő érték: hiba |
| 14 | u16 | fenntartva | Kitöltés/Jövőbeli felhasználásra fenntartva |
| 16 | u32 | payload_len | A következő adatok hossza |

### Adatáramlási entitások

Ez az ábra összeköti az ipc.zig fájlban található protokolldefiníciókat a router.zig fájlban található útválasztási logikával.

### 4.4.3 Időkorlát és hiba kezelése

A router szigorú, 30 másodperces időkorlátot alkalmaz minden bérlői kérésre. Ha egy sandbox nem válaszol ezen időn belül:

1.   A WaitingRequest eltávolításra kerül a függőben lévő elemek listájából
2.   A ProcessTable frissítésre kerül, hogy eltávolítsák az elavult SandboxHandle-t
3.   A sandbox.destroySandbox metódust hívják meg a nem reagáló folyamat leállítására és a cgroupok tisztítására
4.   Az error.QueryTimeout érték kerül visszaadásra az ügyfélnek

### 4.4.4 Állapotmentes bérlők és a WAL-átvitel

A nyilvántartásban „stateless” jelöléssel ellátott bérlők esetében az AGDB egy speciális WAL-átviteli mechanizmust támogat. Ez lehetővé teszi, hogy a sandbox ideiglenes jellegét megőrizze, miközben a Write-Ahead Log (WAL) adatfolyamát egy távoli, állandó végpontra továbbítja.

- RemoteWALTransport: Kezel egy TCP-kapcsolatot (vagy RDMA-csatornát) egy távoli naplógyűjtővel
- Keretformátum: A WAL-rekordokat FrameHeader struktúrákba ágyazzák, amelyek tartalmazzák a tenant_id azonosítót és egy lsn (naplószámozási szám) értéket
- Hatékonyság: Támogatja a kötegelt feldolgozást a DEFAULT_BATCH_CAPACITY beállításon keresztül, valamint a hardveresen gyorsított adatátvitelt az RDMAWALShipper 117 segítségével

### 4.4.5 Wake-Proxy és szunnyadó futásidejű rendszerek

A felhőalapú telepítésekben (pl. OVH VPS) az erőforrás-felhasználás optimalizálása érdekében az agdb-wake-proxy a tétlen futási környezetek életciklus-kezelőjeként működik.

- Észlelés: Ha a fő szerver nem elérhető, a wake-proxy átveszi a bejövő kéréseket
- Aktiválás: Ha a rendszer tétlen állapotban van, az OVH API-n keresztül elindítja a VPS indítóparancsát
- Visszajelzés: Amíg a futtatókörnyezet elindul (általában 30–60 másodpercig tart), egy WAITING_HTML oldalt jelenít meg 503-as állapotkóddal és Retry-After fejléccel.
- Átadás: Amint az isVpsUp igaz értéket ad vissza, a kérés továbbításra kerül az immár aktív futtatási környezetbe

## Hardveres gyorsítás és alacsony szintű alrendszerek

Az AGDB úgy lett kialakítva, hogy a hardverspecifikus funkciók és az alacsony szintű rendszerprimitívek kihasználásával maximalizálja a teljesítményt. Ez a réteg biztosítja az infrastruktúrát a nagy teljesítményű I/O-hoz, a vektorizált számításokhoz és a topológiát figyelembe vevő memóriakezeléshez, így garantálva, hogy az adatbázis hatékonyan skálázható legyen a modern hardverekre.

### Hardver-absztrakciós térkép

Az alábbi ábra bemutatja, hogy a magas szintű adatbázis-műveletek hogyan kapcsolódnak az egyes alacsony szintű hardveres gyorsító modulokhoz.

Hardver-kód leképezés

### SIMD, GPU és számítási gyorsítás

Az AGDB többszintű számítási gyorsítási stratégiát alkalmaz. A szokásos CPU-terhelések esetében a simd modul vektorizált műveleteket biztosít olyan általános feladatokhoz, mint például a tranzakciókban végzett ütközésellenőrzés és a Bloom-szűrővel történő lekérdezések

Nagy számítási terhelés esetén, különösen vektorhasonlósági kereséseknél, az AGDB a GPUContext segítségével integrál egy GPU-gyorsító réteget. Ez az alrendszer kezeli a kernelfelvételt, az eszközmemória-allokációt a GPUArray-en keresztül, és a keresési küszöbértékek elérése esetén a párhuzamos feladatokat – például a cosine_similarity-t – a GPU-ra terheli át.

További részletekért lásd

### NUMA, RDMA és elosztott hardver

A nagyméretű telepítések kezelése érdekében az AGDB tartalmaz alrendszereket a topológiát figyelembe vevő végrehajtáshoz és az alacsony késleltetésű hálózatépítéshez. A NumaTopology modul felismeri a CPU- és memóriaelrendezéseket, hogy az adatok feldolgozása a helyi NUMA-csomóponton történjen, ezzel minimalizálva a processzorok közötti késleltetést.

Elosztott környezetek esetében az AGDB a következőket valósítja meg:

- RDMA: Távoli közvetlen memóriához való hozzáférés az rdma_channel segítségével a zero-copy hálózati kommunikációhoz
- DHTM: Elosztott hardveres tranzakciós memória a csomópontok közötti atomikus műveletek összehangolásához
- HTM: Helyi hardveres tranzakciós memória az optimista párhuzamos feldolgozás vezérléséhez

További részletekért lásd

### I/O, párhuzamos feldolgozási alapelemek és tömörítés

Az AGDB megbízhatóságának és átviteli sebességének alapja a rendszer egyedi I/O- és szinkronizációs primitívjeiben rejlik. A motor a WALIOUringWriter segítségével az io_uring funkciót használja a blokkolásmentes, csővezetékes előreírási napló-megerősítések végrehajtásához

A párhuzamos futást olyan egyedi alapelemekből álló csomag kezeli, amelyeket kifejezetten nagy terhelésű környezetre terveztek:

- PMutex: Hibrid spin/futex mutex a változó zárolási időtartamok mellett is optimális teljesítmény érdekében
- PRWLock: Olvasó-író zár pontos olvasói nyomon követéssel
- SeqLock: Rendkívül gyors olvasási oldali hozzáférést biztosít az állapotpillanatképekhez

Az adatkezelésről az ans modul gondoskodik, amely egy nagy teljesítményű rANS (range Asymmetric Numeral Systems) tömörítőmotort valósít meg a lemezen tárolt adatokhoz

További részletekért lásd

### Az alrendszerek közötti együttműködés áttekintése

Az alábbi ábra bemutatja, hogy ezek az alacsony szintű összetevők hogyan működnek együtt a futásidejű környezetben az adatbázis-motor támogatására.

Futtatási infrastruktúra integrációja

## SIMD, GPU és számítási gyorsítás

Az AGDB hardverspecifikus gyorsítást alkalmaz a számításigényes műveletek – például a vektorhasonlósági keresések, az adatkompresszió és a tranzakciós ütközések felismerése – optimalizálására. Ez egy többszintű megközelítéssel valósul meg: CPU-szintű SIMD (Single Instruction, Multiple Data) utasítások, egy rugalmas GPU számítási környezet a párhuzamos munkaterhelésekhez, valamint egy dedikált, Futhark-alapú számítási program.

### SIMD modul

A SIMD modul vektorizált megvalósításokat biztosít a gyakori adatbázis-műveletekhez, kihasználva az x86_64 AVX2 funkciókat, amennyiben azok rendelkezésre állnak. Különböző bít-szélességű szabványos vektortípusokat határoz meg, például a Vec4u64 (256 bites) és a Vec32u8 típusokat.

### Főbb vektoros műveletek

- Ütközéskeresés: A simdConflictScan 256 bites vektorokat használ az azonosítók (tűk) halmazainak egy cél-tömbhöz (szénakazal) való összehasonlításához. Ezt a TransactionManager használja a tranzakciók közötti írási halmazok metszéspontjainak felismerésére.
- Bloom-szűrők: A BloomFilter256 struktúra egy SIMD-gyorsítású Bloom-szűrőt valósít meg. A simdMightContainBatch függvény lehetővé teszi több kulcs egyidejű ellenőrzését a szűrővel vektoros kiterjesztések és bitműveletek segítségével
- Memóriaműveletek: optimalizált rutinok a memóriák összehasonlításához (simdMemcmp32), a memóriák nullázásához (simdZero) és a vízszintes összegek kiszámításához (simdSum64)
- Előzetes betöltés: A prefetchForRead és a prefetchForWrite parancsok segítségével hardveres utasítások adhatók meg a szekvenciális beolvasás során fellépő cache-hibák minimalizálása érdekében

### GPU-gyorsító réteg

A GPU-gyorsítási réteget a GPUContext kezeli, amely felel a hardver inicializálásáért, a gazdagép és az eszköz közötti memóriakezelésért, valamint a kernelfájlok végrehajtásáért.

### GPUContext és a kernel regisztrációja

A GPUContext inicializálásakor megadható egy dinamikus könyvtár elérési útja, amely GPU-kerneleket tartalmaz.  A GPUContext egy GPUKernelInfo-regisztert tart fenn, amely a karakterlánc-neveket függvénymutatókhoz (GPUKernelFn) rendeli hozzá.

### GPU adattípusok

Az adatok átadása a GPU-nak a GPUValue unió segítségével történik, amely támogatja a skalárokat és a különböző típusú tömböket. A GPUArray(T) struktúra kezeli az adatok életciklusát mind a gazdagépen, mind az eszközön, támogatva a kézi felszabadítást és a hivatkozás-kölcsönzést.

### GPU-keresés integrálása

Az adatbázisrétegben a GPU-gyorsítás egy terhelési küszöbérték alapján aktiválódik. Vektorhasonlósági keresések esetén a rendszer a cosine_similarity számításokat a GPU-ra terhelheti át, ha a vektorok száma meghalad egy előre meghatározott határértéket.

| Komponens | Szerep | Entitás |
| --- | --- | --- |
| Kontextus | Életciklus és regiszter | GPUContext |
| Memória | Gazdagép/eszköz puffer | GPUArray(T) |
| Interfész | Kernel-definíció | GPUKernelInfo |
| Logika | Hasonlósági keresés | cosine_similarity |

### Futhark számítási program

Az AGDB tartalmaz egy Futhark programot (src/compute.fut), amely nagy teljesítményű párhuzamos magokat definiál. Ezeket a magokat lefordítják a GPUContext által használt GPU-könyvtárba.

### Támogatott rendszermagok

A Futhark modul párhuzamos alapelemek átfogó készletét biztosítja:

- Lineáris algebra: Skáláris szorzat, mátrixszorzás és mátrix transzponálás
- Vektoros keresés: euklideszi távolság és koszinusz-hasonlóság
- Statisztika: Átlag, variancia és szórás számítása nagy méretű tömbök esetében
- Adatfeldolgozás: hisztogramok, előtagösszegek (átfutások) és tömbfelosztás
- Jelfeldolgozás: 1D konvolúció és csúszó átlagok

### Számítógépes adatáramlás

Az alábbi ábra bemutatja, hogyan áramlanak az adatok az adatbázis-motorból a GPU-kernelekbe.

Számítási gyorsítási folyamat

### Tenzorműveletek

A Tensor modul magas szintű absztrakciót biztosít a többdimenziós tömbökhöz, és támogatja a SIMD-re optimalizált műveleteket a CPU-n.

- Elrendezés: A tenzorok akár 8 dimenziót is támogatnak egyéni lépésközökkel
- Memória: Igazított (32 bájtos) memóriaterületeket használ az AVX vektorizálás elősegítése érdekében
- Műveletek: Tartalmazza a MatmulComptime-ot a fix méretű mátrixszorzáshoz, valamint a TensorIterator-t a nem összefüggő memóriaterületek bejárásához
- Optimalizálás: A Vec8 (f32x8) vektorszélesség kihasználása érdekében olyan műveletek kerültek megvalósításra, mint a relu, a sigmoid és a tanh

## NUMA, RDMA és elosztott hardver

Az AGDB hardverspecifikus optimalizálásokat alkalmaz annak érdekében, hogy elosztott környezetekben alacsony késleltetésű teljesítményt érjen el. Ide tartozik a topológiát figyelembe vevő memóriakiosztás (NUMA), a közvetlen hardver-hardver kommunikáció (RDMA) és a hardver által támogatott párhuzamos feldolgozás-vezérlés (HTM/DHTM).

### NUMA-topológia és memóriakötés

A NumaTopology rendszer felismeri a gazdagép fizikai elrendezését, hogy optimalizálja a memóriához való hozzáférési mintákat és minimalizálja a processzorok közötti késleltetést.

### Topológia-felismerés

Linux-rendszereken a motor átvizsgálja a /sys/devices/system/node/ könyvtárat, hogy létrehozza a rendelkezésre álló NUMA-csomópontok, a hozzájuk tartozó CPU-magok és a memóriastatisztikák térképét. Különösen azonosítja a nagy sávszélességű memóriával (HBM) rendelkező csomópontokat, amelyekre jellemző, hogy memóriával rendelkeznek, de nincs hozzájuk kapcsolt CPU

### Memória-beállítások

Az AGDB a set_mempolicy és az mbind parancsok segítségével számos Linux-memóriapolitikát támogat

- MPOL_BIND: A memóriát kizárólag egy meghatározott csomópontcsoporton osztja el.
- MPOL_INTERLEAVE: A sávszélesség maximalizálása érdekében a forráskiosztást a csomópontok között felosztja.
- MPOL_PREFERRED: Először egy adott csomóponton próbálja meg az allokációt, de ha az megtelt, akkor más csomópontokra vált át.

### Végrehajtó szervek

| Entitás | Szerepkör |
| --- | --- |
| NumaTopology | A rendszer hardverfelépítésének fő nyilvántartása  |
| CpuSet | A csomópont CPU-hoz való kötődését leíró bitmaszk  |
| NumaNode | Nyomon követi a csomópontok azonosítóit, memóriakapacitását és a többi csomóponttól való távolságukat  |

### RDMA és alacsony késleltetésű hálózatok

Az AGDB a Remote Direct Memory Access (RDMA) technológiát alkalmazza, amely lehetővé teszi a csomópontok számára, hogy egymás memóriájába közvetlenül olvassanak vagy írjanak anélkül, hogy a távoli oldalon az operációs rendszer kernele vagy a CPU közreműködne.

### RDMA architektúra

A rendszer mind a hardveres protokollokat (Infiniband/RoCE), mind a szoftveres TCP-visszaesést támogatja

Hardveres entitás-leképezés

### RDMA-csatorna (rdma_channel.zig)

Az RDMAChannel magas szintű absztrakciót biztosít az adatok (például a WAL-rekordok) RDMA-n keresztüli továbbításához.

- FlightTracker: Az RDMAFlightEntry segítségével kezeli a repülés közbeni RDMA-műveleteket, biztosítva azok sorrend szerinti végrehajtását és az LSN-nyomkövetést
- ChannelEndpoint: A távoli csomópont számára a helyi memóriaterület eléréséhez szükséges qp_num, rkey és mr_addr értékeket tartalmazza

### Elosztott hardveres tranzakciós memória (DHTM)

A DHTM kiterjeszti a helyi hardveres tranzakciós memóriát (HTM), hogy több fizikai csomópont között koordinálja a tranzakciókat.

### A tranzakció életciklusa

1.   Helyi fázis: A tranzakció végrehajtásának kísérlete CPU-szintű HTM használatával
2.   Előkészítési szakasz: A koordinátor PrepareMsg üzenetet küld minden résztvevőnek
3.   Ütközésfelismerés: A csomópontok SIMD-gyorsítású szkennelést (simdConflictScan) alkalmaznak az írási/olvasási halmazok közötti átfedések ellenőrzésére
4.   Véglegesítés/Megszakítás: Ha minden résztvevő visszaigazolja, a koordinátor véglegesíti a műveletet; ellenkező esetben visszavonja azt

### DHTM adatstruktúrák

- DHTMVersionRecord: 32 bájtos rekord, amelyet bizonyos memóriacímek zárolására és verziókezelésére használnak
- DHTMTransaction: Nyomon követi a globális tranzakció írási és olvasási halmazát

### Hardveres tranzakciós memória (HTM)

Az AGDB az x86-64 Restricted Transactional Memory (RTM) technológiát használja az optimista párhuzamos feldolgozás vezérléséhez a helyi műveletek során.

### HTM-alapelemek

A motor az xbegin, xend és xabort utasításokat burkolja.  Egy runWithHTM segédfüggvényt biztosít, amely automatikusan kezeli az újrapróbálkozásokat, és standard mutexre vált át, ha a hardveres tranzakció ütközések vagy kapacitási korlátok miatt ismétlődően sikertelen

A HTM végrehajtási folyamata

### TSC és hardveres véletlenszám-generátor

Az AGDB az időzítés és a biztonság biztosításához nagy felbontású hardveres számlálókra és entrópiaforrásokra támaszkodik.

### Időbélyeg-számláló (TSC)

- rdtsc / rdtscp: CPU-ciklus-pontos időzítést biztosít
- TscSequencer: a TSC-t egy atomi számlálóval kombinálja, hogy monoton, globálisan egyedi 64 bites azonosítókat generáljon

### Hardveres véletlenszám-generátor

A HardwareRng struktúra az RDRAND utasítást használja a kiváló minőségű entrópia előállításához. Amennyiben a hardveres utasítás nem áll rendelkezésre, a struktúra tartalmaz egy tartalék megoldást, amely a rendszeridő és a TSC kombinációjával inicializált PRNG-t használ.

### Gyorsítótár- és igazítási segédprogramok

A topology.zig modul olyan segédprogramokat tartalmaz, amelyek biztosítják, hogy az adatstruktúrák a CPU gyorsítótár-hierarchiájához legyenek optimalizálva.

- CacheLinePadded: Egy típust úgy csomagol be, hogy az egy teljes cache-sort (64 bájt) foglaljon el, így megakadályozva a magok közötti „hamis megosztást”
- WorkStealDeque: Zármentes, topológiát figyelembe vevő, kétirányú sor, amelyet a feladatok magok közötti elosztására használnak
- fieldPackingAnalysis: Egy fordítási idejű segédprogram, amely kiszámítja egy struct csomagolási hatékonyságát és az elpazarolt bájtokat

## I/O, párhuzamos feldolgozási alapelemek és tömörítés

Ez az oldal az AGDB alacsony szintű infrastruktúráját mutatja be, különös tekintettel az io_uring segítségével megvalósított nagy teljesítményű I/O-ra, a tartós tárolásra tervezett, megbízható, egyedi párhuzamos feldolgozási primitívek halmazára, valamint a rANS-alapú tömörítő motorra.

### 1. Nagy teljesítményű I/O az io_uring segítségével

Az AGDB a Linux io_uring funkciót használja a nagy átviteli sebességű, blokkolásmentes I/O megvalósításához, különösen a Write-Ahead Log (WAL) műveletek esetében. Az IOUring struktúra biztonságos Zig-burkolatot biztosít a rendszermag benyújtási és befejezési sorai köré.

### io_uring architektúra

A megvalósítás három memóriaterületet képez le a felhasználói tér és a rendszermag-tér között: a beküldési sor (SQ), a befejezési sor (CQ) és a beküldési sorbejegyzések (SQE) tömb

| Komponens | Leírás |
| --- | --- |
| io_uring_sqe | Beküldési sorba állított bejegyzés: tartalmazza az opkódot, az fájlleírószámot, az eltolást és a puffer címét  |
| io_uring_cqe | Befejezési sorba állított bejegyzés: tartalmazza az eredményt és a nyomon követéshez szükséges user_data adatot  |
| IOUring.submit() | Belép a kernelbe a feldolgozásra váró SQE-k feldolgozása érdekében  |
| IOUring.waitCQE() | Addig várakozik, amíg legalább egy befejezési bejegyzés elérhetővé válik  |

### I/O absztrakciós réteg

Az io.zig modul olyan determinisztikus hash-funkciókat biztosít, amelyeket az adatok integritásának biztosítására és az I/O-pufferek közötti elosztására használnak.

- stableHash: egy 64 bites hash-függvény, amelyet az adatok és a tárolóblokkok közötti konzisztens leképezéshez használnak
- combineHashes: Két hash-értéket egyesít egy 0x9E3779B97F4A7C15 bitkeverő konstans segítségével

### 2. Párhuzamos feldolgozási alapelemek

Az AGDB a src/concurrency.zig fájlban egy sor egyedi szinkronizációs primitíveket valósít meg. Ezeket úgy tervezték, hogy „tartós tárolásra alkalmasak” legyenek, vagyis tartalmaznak mágikus számokat és ellenőrző összegeket a megosztott memóriaterületeken fellépő sérülések észleléséhez.

### Hibrid mutex (PMutex)

A PMutex hibrid megközelítést alkalmaz: először egy beállítható számú cikluson (spin_count) keresztül megpróbálja a spin-módot, majd ha ez nem sikerül, a Linux Futex-alapú várakozási módra vált át

Főbb jellemzők:

- Mágikus/verzióellenőrzés: Használat előtt ellenőrzi a zárolási struktúrát
- Tulajdonoskövetés: Az aktuális tulajdonos szálazonosítóját tárolja a holtpontok felismerése érdekében
- Állapotgép: Átmenetek az STATE_UNLOCKED (0), STATE_LOCKED (1) és STATE_CONTENDED (2) állapotok között

### Olvasó-író zár (PRWLock)

A PRWLock egyszerre több olvasót vagy egyetlen írót támogat. Tartalmaz egy szál-helyi tárolón alapuló olvasói nyomonkövetést, amely megakadályozza, hogy egy szál több olvasási zárat szerezzen ugyanazon erőforrásra, és ezzel erőforráshiányt okozzon.

### A Guard-regiszter és az RAII

Az AGDB RAII-típusú őrzőket (MutexGuard, ReadGuard, WriteGuard) használ annak biztosítására, hogy a zárolások akkor is felszabaduljanak, ha egy függvény idő előtt visszatér vagy hiba történik. Ezeket az őrzőket egy globális guard_registry_entries táblában regisztrálják, hogy a rendszer ellenőrizhesse a fennálló zárolásokat

### Párhuzamos feldolgozás és entitás-leképezés

Az alábbi ábra bemutatja a szinkronizálás általános fogalmait és azok megvalósító struktúráit a src/concurrency.zig fájlban.

Szinkronizációs infrastruktúra

### 3. SeqLock a gyors olvasási hozzáférés érdekében

Az olyan adatok esetében, amelyeket gyakran olvasnak, de ritkán frissítenek (például tranzakciós metaadatok vagy rendszerstatisztikák), az AGDB a SeqLock-ot használja. Ez lehetővé teszi az olvasók számára, hogy az adatokhoz hozzáférjenek anélkül, hogy hagyományos zárolást szereznének vagy atomi írási műveleteket hajtanának végre, így elkerülve a cache-sorok közötti ugrálást.

- SeqLockData(T): A T típus védelmét szolgáló generikus burkoló. Az olvasók pillanatfelvételt készítenek, majd ellenőrizik, hogy a sorszámláló nem változott-e
- SeqLockBytes: Nyers bájtpufferek védelmére optimalizálva, gyakran használják állapotsorokhoz vagy sorosított fejlécekhez

### 4. Tömörítés: rANS Engine

Az AGDB egy aszimmetrikus számrendszerű (ANS) tömörítő motort valósít meg, mégpedig annak tartományváltozatát (rANS). Ez az aritmetikai kódoláshoz hasonló tömörítési arányt biztosít, de sebességét tekintve inkább a Huffman-kódoláshoz áll közel.

### A megvalósítás elemei

1.   SymbolStats: Elemezi a bemeneti adatokat a szimbólumok gyakoriságának és a kumulatív gyakoriságok kiszámításához, ANS_SCALE (2^12) skálára átszámítva
2.   AliasTable: A dekóder által használt keresőtábla, amely a kicsomagolás során O(1) időkomplexitású szimbólum-visszakeresést tesz lehetővé
3.   RansEncoder: A szimbólumokat 64 bites állapotba kódolja. Amikor az állapot meghaladja az ANS_UPPER_BOUND értéket, a bájtokat a kimeneti adatfolyamba továbbítja
4.   RansDecoder: Visszafordítja a folyamatot úgy, hogy a bájtokat feldolgozza, és az AliasTable segítségével az állapotot visszaalakítja szimbólumokká

### Adatáramlás: Tömörítési folyamat

Az alábbi ábra bemutatja, hogyan haladnak át az adatok a tömörítő egységeken.

Tömörítési adatáramlás

### WCBuffer (írás-összevonó puffer)

A tömörített adatok állandóságának optimalizálása érdekében az AGDB a WCBuffer-t használja (amire az ans.zig kontextus és az igazított írási műveletek utalnak). Ez biztosítja, hogy a tárolóeszközre történő írási műveletek a hardveres gyorsítótár-sorokhoz vagy oldalhatárokhoz igazodjanak, csökkentve ezzel a részleges írási műveletekkel járó terhelést.

## Biztonság

Az AGDB biztonsági architektúrája úgy lett kialakítva, hogy megvédje a tárolt adatokat, biztosítsa a tartós halom integritását, és kihasználja a hardver által támogatott bizalmi mechanizmusokat. Ez az oldal átfogó áttekintést nyújt ezekről az összetevőkről; a részletes magyarázatok a témának szentelt aloldalakon találhatók.

A legfontosabb biztonsági funkciók közé tartozik az érzékeny adatok megbízható titkosítása, a Trusted Platform Modules (TPM) modulokkal való integráció a hardveralapú bizalom biztosítása érdekében, valamint egy átfogó integritás-ellenőrző rendszer a jogosulatlan módosítások felismerésére.

### Titkosítás és kulcskezelés

Az AGDB erős kriptográfiai primitíveket alkalmaz a tárolt adatok védelmére. A SecurityManager irányítja a titkosítási és visszafejtési műveleteket, támogatva mind az AES-GCM, mind a ChaCha20-Poly1305 AEAD (Authenticated Encryption with Associated Data) algoritmusokat. Kezelése alá tartoznak az EncryptionKeys kulcsok, beleértve a főkulcsot is, és a kulcsderivációt HKDF-SHA256 segítségével végzi. A kriptográfiai egyediség biztosítása érdekében nonce-generálási stratégiák kerültek bevezetésre. Azokban a környezetekben, ahol a biztonság nem elsődleges fontosságú, a titkosítás kikapcsolható.

A titkosítási algoritmusokkal, a kulcsderivációval, a nonce-generálással, a főkulcs-rotációval és az EncryptedRegion bináris formátummal kapcsolatos részleteket lásd

## A SecurityManager titkosítási és visszafejtési folyamatának forrásai:

### TPM-integráció és integritás-ellenőrzés

Az AGDB a Trusted Platform Modules (TPM) modulokkal integrálódik, hogy hardveralapú bizalmi láncot hozzon létre. A TPM2Interface (amelyet az src/tpm.c támogat) lehetővé teszi az érzékeny adatok lezárását és feloldását meghatározott PCR (Platform Configuration Register) állapotokhoz, biztosítva, hogy az adatokhoz csak akkor lehessen hozzáférni, ha a rendszer konfigurációja megegyezik egy előre meghatározott biztonsági állapotgal. Ez megakadályozza a rendszerindítási folyamat vagy a rendszer szoftverének meghamisítását.

Ezen felül az IntegrityVerifier komponens biztosítja a tartós halom integritását. Az adatlapokból egy Merkle-fát épít fel, amelyben az egyes lapok SHA-256 hash-értéke egyetlen gyökér-hash-értékhez járul hozzá. Ez a merkle_root az egész adattár integritásának tömör és ellenőrizhető ábrázolásaként szolgál. A bizonyítékalapú ellenőrzés lehetővé teszi az egyes oldalak hatékony ellenőrzését anélkül, hogy az egész fát újra kellene számolni.

A TPM2Interface olyan funkcióinak részletes leírását, mint a PCR-leolvasás lezárása és feloldása (readPCR), valamint az IntegrityVerifier Merkle-fa felépítése (getMerkleRoot), lásd

## A TPM és az integritás-ellenőrzési architektúra forrásai:

## Titkosítás és kulcskezelés

A SecurityManager az AGDB-n belüli adatbiztonságért és -integritásért felelős központi komponens. Többszintű titkosítási stratégiát biztosít, amely a memóriaterületek magas szintű, társított adatokkal ellátott hitelesített titkosításától (AEAD) a gyorsítótár-sorok állandóságát biztosító alacsony szintű, szektor alapú AES-XTS titkosításig terjed. A rendszer integrálja a hardver által támogatott entrópiát és gyorsítást, miközben tartalékmechanizmusokat biztosít a nem biztonságos vagy nem gyorsított környezetek számára.

### A SecurityManager architektúrája

A SecurityManager kezeli a titkosítási kulcsok életciklusát és koordinálja a kriptográfiai műveleteket. Támogat egy globális kapcsolót a titkosítás engedélyezéséhez vagy letiltásához, ami lehetővé teszi a teljesítményorientált telepítéseket megbízható környezetekben.

### Főbb összetevők

- Fő kulcs: Az a fő titkosítási kulcs, amelyet alkulcsok levezetésére vagy általános régiók titkosítására használnak
- Régiókulcsok: A key_id alapján indexelt kulcsokból álló hash-térkép, amely részletes hozzáférés-vezérlést és rotációt tesz lehetővé
- Hardveres gyorsítás: A hasAESNI() függvény segítségével automatikusan felismeri az AES-NI támogatást az AES-GCM és az AES-XTS teljesítményének optimalizálása érdekében
- Biztonságos nullázás: A deinit függvényt valósítja meg, amely a crypto.utils.secureZero segítségével biztonságosan törli a főkulcsot a memóriából

### Adatáramlás: Titkosítási kérelem

Az alábbi ábra bemutatja a folyamatot, amikor a rendszer titkosított régiót kér.

A titkosítási kérelem feldolgozásának folyamata

### Kriptográfiai algoritmusok

Az AGDB modern, nagy teljesítményű alapelemeket használ a különböző tárolási igények kielégítésére:

| Alkalmazási eset | Algoritmus | Megvalósítás |
| --- | --- | --- |
| Hitelesített tárolás | AES-256-GCM | crypto.aead.aes_gcm.Aes256Gcm |
| Alternatív AEAD | ChaCha20-Poly1305 | crypto.aead.chacha20.ChaCha20Poly1305 |
| Cache-Line / Lemez | AES-XTS | encryptCacheLine |
| Kulcsgenerálás | HKDF-SHA256 | deriveKey |

### AES-XTS cache-sor titkosítás

Azoknál a tartós tárolási megoldásoknál, amelyek szektor alapú beállításokat igényelnek (például gyorsítótár-sorok vagy lemezszektorok), az AGDB az AES-XTS algoritmust alkalmazza. Ez biztosítja, hogy ugyanazon szöveg különböző eltolásokkal történő titkosítása eltérő titkosított szöveget eredményezzen.

- Tweak generálás: Galois-tér szorzásával (gfMulX) kapott, szektor alapú tweak-et használ
- Cache-sor titkosítás: Az encryptCacheLine függvény kifejezetten a nagy teljesítményű perzisztencia-rétegekhez alkalmazza az XTS módot

### Kulcskezelés és kulcsgenerálás

### A kulcs életciklusa

A kulcsokat az EncryptionKey struktúra képviseli, amely nyomon követi a nyers bájtokat, az algoritmust, egy egyedi key_id azonosítót és a létrehozás időbélyegét

1.   Inicializálás: A kulcsok a crypto.random.bytes segítségével kerülnek generálásra
2.   Kivonás: A `deriveKey` függvény az HKDF-SHA256 algoritmust használja alkulcsok létrehozására a fő titkos kulcsból és a HKDFParams paraméterekből (só és információs karakterlánc)
3.   Rotáció: A SecurityManager támogatja a master_key frissítését. A deinit során az összes érzékeny kulcsadatot nullázzák

### Egyedi szám generálása

Az AGDB két stratégiát alkalmaz a nonce-ok generálására az újrafelhasználás megakadályozása érdekében:

- Szabvány: Egy atomi nonce_counter-t véletlenszerű bájtokkal kombinál
- Hardveres támogatás: Az RDRAND-ot (a tsc.zig-en keresztül) és a Blake3 hash-algoritmust használja a magas entrópiájú nonce-ok generálásához érzékeny alkalmazásokban.

### Bináris formátum: EncryptedRegion

Amikor az adatokat tárolás vagy továbbítás céljából titkosítják, azokat az EncryptedRegion struktúrába ágyazzák.

A titkosított régió memóriaterülete

| Mező | Típus | Leírás |
| --- | --- | --- |
| titkosított szöveg | []u8 | A titkosított adatcsomag |
| nonce | [12]u8 | A titkosítási művelethez használt egyedi IV |
| címke | [16]u8 | Integritásellenőrző címke (MAC) |
| key_id | u64 | A használt EncryptionKey azonosítója |
| algoritmus | u8 | Enum-érték (AES-GCM vagy ChaCha20) |

### Hardveres bizalom és TPM-integráció

A biztonsági modell a TPM2Interface-en keresztül kiterjed a hardver által támogatott megbízhatóságra (C nyelven valósítva meg a TSS2-kompatibilitás érdekében).

- PCR-lezárás: A tpm2_seal függvény lehetővé teszi az érzékeny adatok (például a főkulcs) „lezárását” meghatározott platformkonfigurációs regiszterekbe (PCR-ekbe)
- Feloldás: Az adatok csak akkor dekódolhatók (tpm2_unseal), ha a rendszer állapota (PCR-ek) megegyezik a lezáráskori állapotgal
- TCTI-kommunikáció: Támogatja a TPM-mel való kommunikációt a /dev/tpm0 eszközön vagy a környezetben definiált TCTI-betöltőkön keresztül

TPM-interakciós logika

## TPM-integráció és integritás-ellenőrzés

Az AGDB biztonsági architektúrája a Trusted Platform Module (TPM 2.0) hardvert és kriptográfiai Merkle-fákat használ a tartós adathalmaz titkosságának és integritásának biztosítására. Ez a rendszer védelmet nyújt a tárolt adatok jogosulatlan módosítása ellen, és garantálja, hogy az adatbázis kizárólag ellenőrzött futtatási környezetben működjön.

### TPM 2.0 interfész

A TPM2Interface a TCG Software Stack (TSS2) segítségével hidat képez a Zig futásideje és a fizikai TPM-hardver között. C-nyelvű wrapperként (src/tpm.c) valósult meg, amely a libtss2-esys és a libtss2-tctildr könyvtárakhoz kapcsolódik.

### Főbb összetevők és adatáramlás

- TCTI kommunikációs útvonal: Az interfész inicializálja a TPM parancsátviteli interfészt (TCTI) a hardvereszközzel (általában /dev/tpm0) vagy egy szimulátorral való kommunikációhoz
- PCR-leolvasás: Lehetővé teszi a rendszer számára a platformkonfigurációs regiszterek (PCR-ek) leolvasását a rendszer firmware-jének aktuális állapotának és a rendszerindítási sorrendnek az ellenőrzése céljából
- Lekötés és feloldás: Az érzékeny adatok (például a SecurityManager főkulcsa) leköthetők meghatározott PCR-értékekhez. Ez biztosítja, hogy az adatok csak akkor legyenek feloldhatók, ha a rendszer megbízható állapotban van

### TPM interakciós diagram

Az alábbi ábra bemutatja, hogyan kezeli a tpm2_context_t a TPM-munkamenet életciklusát.

- (a tpm2_context_t típus definíciója)
- (tpm2_init megvalósítás)
- (tpm2_read_pcr megvalósítás)
- (tpm2_seal megvalósítás)

### Integritás-ellenőrző rendszer

Az IntegrityVerifier biztosítja, hogy a PersistentHeap-et külső szereplők ne manipulálhassák. Oldalszintű hash-stratégiát alkalmaz, amelyet Merkle-fával kombinál, így egyetlen megbízható gyökérpontot biztosít.

### Merkle-fa felépítése

1.   Oldal-hasholás: A halom rögzített méretű oldalakra van felosztva. Minden oldal SHA-256 algoritmussal kerül hash-elésre.
2.   Faépítés: Ezek az oldal-hashértékek alkotják a Merkle-fa leveleit. A belső csomópontokat úgy számítjuk ki, hogy a gyermekcsomópontok hashértékeinek összefűzését hash-eljük.
3.   Bizalmi gyökér: A végső merkle_root a teljes halom állapotát jelenti. Ez a gyökér tárolható a TPM nem felejtő (NV) RAM-jában, vagy aláírható egy TPM által védett kulccsal.

### Ellenőrzési logika

Amikor egy oldalt beolvas a PersistentHeapből, az IntegrityVerifier újraszámolja annak hash-értékét, és összehasonlítja a kriptográfiai bizonyítékot az aktuális Merkle-gyökkel. Ha a hash-értékek nem egyeznek, a rendszer módosítást észlel, és megakadályozza a tranzakció folytatását.

### Kód-entitás leképezés: integritás és biztonság

Ez az ábra a magas szintű biztonsági koncepciókat a kódbázis konkrét struktúráihoz és függvényeihez rendeli hozzá.

### Főbb funkciók

| Funkció | Hely | Leírás |
| --- | --- | --- |
| SecurityManager.init |  | Inicializálja a biztonsági alrendszert, felismerve a hardveres gyorsítást (AES-NI). |
| SecurityManager.encrypt |  | A puffert AES-GCM vagy ChaCha20-Poly1305 algoritmussal titkosítja. |
| tpm2_seal |  | Az adatpuffert a TPM elsődleges kulcsával egy PCR-maszkhoz kapcsolja. |
| tpm2_unseal |  | Csak akkor oldja fel az adatok lezárását, ha a PCR állapota megegyezik a lezáráskori beállítással. |

- (SecurityManager típus)
- (EncryptionAlgorithm enum)
- (tpm2_digest_t típus)

### Biztonsági adatáramlás

A TPM és az integritás-ellenőrzés integrálásával egy biztonságos indításhoz hasonló lánc jön létre az adatbázis-heap számára:

1.   Futtatás kezdete: A SecurityManager megkísérli letölteni a főkulcsot.
2.   TPM-zár feloldása: Ha a kulcs lezárva van, a TPM2Interface meghívja a tpm2_unseal függvényt. Ez csak akkor sikerül, ha a rendszer PCR-jei (amelyek a firmware és az operációs rendszer integritását jelzik) érvényesek
3.   Heap-hozzáférés: Amint a PersistentHeap betölti az oldalakat, az IntegrityVerifier összehasonlítja az oldal hash-értékét a Merkle-fával.
4.   Visszafejtés: Ha az integritás ellenőrzése sikeres, a SecurityManager a lezárt főkulcs segítségével visszafejteti az oldal tartalmát

### Hardveres gyorsítás

A SecurityManager az inicializálás során a hasAESNI() ellenőrzés segítségével automatikusan felismeri az AES-NI támogatását, lehetővé téve ezzel az aes_gcm algoritmus nagy teljesítményű kriptográfiai műveleteinek végrehajtását

- (Főkulcs generálása)
- (AES-GCM visszafejtési útvonal)
- (TPM elsődleges kulcs azonosítójának megszerzése)

## JSON, sorosítás és segédmodulok

Ez az oldal az AGDB magasabb szintű komponenseit támogató alapvető segédmodulokat mutatja be. Ide tartozik a nagy teljesítményű HTTP-kommunikációhoz kifejlesztett egyedi JSON-megvalósítás, a biztonságot és a párhuzamos futást szolgáló speciális memóriakezelési primitívek, valamint a motor- és a felhőrétegekben egyaránt használt közös adatstruktúrák.

### JSON megvalósítás

Az AGDB egy egyedi JSON-elemzőt és -formázót használ, amely a  A modul kifejezetten a Cloud API követelményeinek teljesítésére, valamint az agdb-cloud szerver és a bérlői homokozók közötti folyamatok közötti kommunikáció kezelésére lett kifejlesztve.

### Adatmodell

A Value unió a JSON típusrendszert képviseli, az objektumok hatékony tárolásához a std.StringArrayHashMapUnmanaged osztályt használja

| Típus | Zig-ábrázolás | Leírás |
| --- | --- | --- |
| null | null_value | JSON null-literál. |
| bool | bool_value: bool | Boole-érték (igaz/hamis). |
| szám | int_érték: i64 / float_érték: f64 | Támogatja mind az egész számokat, mind a lebegőpontos számokat. |
| karakterlánc | karakterlánc: []const u8 | UTF-8 kódolású karakterlánc. |
| tömb | tömb: []Érték | Az Értékek rekurzív szelet |
| objektum | objektum: ObjectEntries | Karakterláncok és értékek közötti kulcs-érték táblázat. |

### A megvalósítás részletei

- Elemzés: A Parser struktúra egy rekurzív lefelé haladó elemzőt valósít meg, amely kezeli a szóközök kihagyását, az escape-szekvenciákat és az UTF-8-es érvényesítést
- Memóriakezelés: Mivel a JSON-objektumok gyakran tartalmaznak egymásba ágyazott memóriaterületeket, a Value típus egy deinit metódussal rendelkezik a memória rekurzív felszabadításához. Ezenkívül támogatja a mélyklónozást a clone metódus segítségével.
- Típusátalakítás: Az asInt, asFloat és asBool segédmódszerek biztonságos hozzáférést biztosítanak az alapul szolgáló adatokhoz, szükség esetén automatikus típusátalakítással az egész és a lebegőpontos típusok között

### A JSON-elemzés folyamata

Az alábbi ábra szemlélteti a nyers bájtok Value-fává történő átalakítását.

A JSON-elemzés menete

### Memória segédprogramok (mem_utils)

A mem_utils modul speciális memóriakezelőket és alacsony szintű memóriaprimitíveket biztosít, amelyek az AGDB teljesítményi és biztonsági követelményeihez vannak optimalizálva.

### Speciális allokátorok

1.   Arena: Fix méretű, szálbiztos memóriaterület, amely gyors lineáris memóriakiosztást támogat.  Tartalmaz egy secureReset funkciót, amely a memóriát újrahasznosítás előtt törli
2.   ArenaAllocator: Az Arena rugalmasabb változata, amely úgy bővíthető, hogy további puffereket rendel hozzá egy szülő allokátortól

### Alapszintű primitívek

- secureZeroMemory: Az adatok memóriából való törlésére szolgál (például API-kulcsok vagy lezárt TPM-adatok esetében), hogy megakadályozza az adatok kiszivárgását az objektum felszabadítása után
- Ellenőrző függvények: Az alignForwardChecked és az isPow2 függvények biztosítják, hogy a memóriacímek megfeleljenek a hardverkövetelményeknek, különösen a SIMD és az O_DIRECT I/O esetében
- Memória konfiguráció: Meghatározza a rendszerre vonatkozó állandókat, például a PAGE_SIZE-t (4 KB vagy 16 KB, az architektúrától függően) és a CACHE_LINE_SIZE-t (128 bájt)

A src/types.zig fájl a motor egészében használt általános adatstruktúrákat és matematikai típusokat határozza meg, beleértve a keresési rangsorolást és a valószínűségi indexelést is.

### Bitkészlet

Dinamikus bitkészlet-megvalósítás, amelyet a dokumentumok jelenlétének nyomon követésére a keresési eredmények között, illetve a szabadlisták kezelésére használnak

- Tárolás: Az []u64-et használja alapul szolgáló szótárolóként
- Műveletek: Támogatja a beállítást, törlést, lekérdezést, valamint a @popCount beépített parancs segítségével végrehajtható nagy teljesítményű számlálási műveletet

### Fixpontos aritmetika (Fixed32_32)

A különböző hardverarchitektúrák közötti determinisztikus számításokhoz (ahol a lebegőpontos viselkedés kissé eltérhet) az AGDB egy 64 bites fixpontos típust használ, amelynek 32 bitje a tizedes részre vonatkozik

- Pontosság: Állandó eredményeket biztosít az értékelés és a rangsorolás során

### PRNG (pszeudo-véletlenszám-generátor)

Gyors, nem kriptográfiai, XorShift-alapú generátor, amelyet belső szimulációkhoz, mintavételhez és nem biztonsági szempontból kritikus véletlenszerűsítéshez használnak

### Rendszerelemek leképezése

Az alábbi ábra a magas szintű rendszerkoncepciókat ábrázolja a kódbázisban található konkrét megvalósításaikkal együtt.

Közüzemi szervezetek leképezése

### Hibakezelés

Az AGDB egy központi hibakészletet határoz meg, amely az összes modulra kiterjedő általános hibamódokat fedi le, ideértve a következőket:

- Memóriahibák: OutOfMemory, InvalidAlignment, InvalidSize.
- Adatintegritás: Érvénytelen adat, Érvénytelen verzió, Alakzateltérés.
- Stream/IO: EndOfStream, OutOfBounds.

Ez az egységes hiba-készlet lehetővé teszi a hibák következetes továbbítását és kezelését a C++-szerű alapmotor logikája és a magas szintű Zig felhőszolgáltatások közötti határfelületen.

## Szótár

Ez a szószedet az AGDB kódbázisában használt szakszavakat, rövidítéseket és szakterületi fogalmakat tartalmazza. Az AGDB egy nagy teljesítményű, állandó adatbázis-motor, amelyet hibrid kereséshez (BM25 + vektor) és többfelhasználós felhőalapú telepítésekhez terveztek.

### A tárolás alapfogalmai

### Tartós halom

Az a alapvető tárolási absztrakció, amely a memóriába leképezett fájlt összefüggő halomként kezeli. A tartósság érdekében hardver szintű optimalizálásokat biztosít, ideértve a gyorsítótár-sorok ürítését és a módosított oldalak nyomon követését.

- Megvalósítás: a PersistentHeap struktúra a
- Főbb funkciók: init

### PersistentPtr / RelativePtr

A PersistentHeap-ben tárolt adatstruktúrákhoz használt, címfüggetlen mutatók. A PersistentPtr egy pool_uuid-t és egy eltolást használ annak biztosítására, hogy a mutatók a folyamat újraindításakor vagy különböző memóriatér-térképzési címek esetén is érvényben maradjanak.

- Megvalósítás: PersistentPtr a
- Adatáramlás: A PersistentAllocator használja a kiosztott blokkokhoz tartozó azonosítók visszaadására

### Táblaelosztás

Kis objektumok kezelésére alkalmazott memóriakezelési stratégia, amelyben a memóriát fix méretű „elemekből” álló „tömbökre” osztják fel a töredezettség és a metaadatok okozta terhelés minimalizálása érdekében.

- Megvalósítás: CacheLinePackedSlab a
- Állandók: NUM_SIZE_CLASSES (32) és MAX_SMALL_SIZE (4096 bájt)

### Párhuzamos feldolgozás és tranzakciók

### Az ACID garanciái

Az AGDB a Write-Ahead Logging (WAL) és a tranzakciókezelő kombinációjával biztosítja az atomitást, a konzisztenciát, az elszigeteltséget és a tartósságot.

- WAL megvalósítás: a WAL struktúra a
- Tranzakciókezelő: TransactionManager a

### HTM (hardveres tranzakciós memória)

Az AGDB az optimista párhuzamosítás-vezérléshez az x86-64 RTM (Restricted Transactional Memory) utasításokat (XBEGIN, XEND, XABORT) használja. Megkísérli a kritikus szakaszokat zár nélkül végrehajtani, és csak ütközés vagy kapacitáshiány miatti megszakítás esetén tér vissza a hagyományos mutex használatához.

- Megvalósítás: htm.zig
- Főbb funkciók: htmBegin

### DHTM (Elosztott hardveres tranzakciós memória)

A HTM kiterjesztése többcsomópontos koordinációra, amely kétfázisú protokollt és verziórekordokat használ a hálózati határokon átnyúló ütközések felismerésére.

- Megvalósítás: DHTMTransaction a
- Ütközésfelismerés: checkConflictSIMD

### Keresés és indexelés

### BM25 index

A keresőmotorok által használt rangsorolási funkció, amely a dokumentumok adott keresési lekérdezéshez való relevanciáját becsüli meg. Az AGDB TF-IDF-pontozással ellátott fordított indexet valósít meg.

- Megvalósítás: Bm25Index a
- Főbb funkciók: addDocument

### Vektorindex

Magas dimenziójú vektorbeágyazásokhoz készült index, amely támogatja az olyan hasonlósági mérőszámokat, mint a koszinusz, a belső szorzat és az euklideszi távolság.

- Megvalósítás: VectorIndex a

### ANS (aszimmetrikus számrendszerek)

Az adatbázison belüli adatkompresszióhoz használt nagy teljesítményű entrópia-kódolási módszer, amely kifejezetten egy tartományalapú változatot (rANS) alkalmaz.

- Megvalósítás: RansEncoder és RansDecoder
- Adatstruktúra: AliasTable a szimbólumok gyors, O(1) időkomplexitású dekódolásához

### Felhőalapú szolgáltatások és többfelhasználós rendszer

### Kísérleti környezet

Egyetlen bérlő számára biztosított elszigetelt futtatási környezet. Az AGDB-Cloud a Linux névterek (PID, Network, Mount) és a Cgroups v2 segítségével korlátozza az erőforrás-használatot és biztosítja a biztonságot.

- Végrehajtás: spawnTenantSandbox a
- Elszigetelési jelzők: CLONE_NEWPID, CLONE_NEWNS, CLONE_NEWNET

### Rendszerleíró adatbázis

Az a központi vezérlőréteg-összetevő, amely a bérlői metaadatokat, az API-kulcsokat és a fájlrendszer-útvonalakat kezeli.

- Végrehajtás: Nyilvántartás a
- Bérlői adatlap: TenantRecord struktúra

### Rendszerarchitektúra-ábrák

### A természetes nyelvtől a kódig: a tárolási réteg

Az alábbi ábra összehasonlítja az általános tárolási koncepciókat az azokat megvalósító konkrét struktúrákkal és fájlokkal.

### A természetes nyelvtől a kódig: többfelhasználós rendszer

Az alábbi ábra bemutatja a többfelhasználós rendszer követelményeinek és a felhőréteg-koordinációs logikának az összefüggéseit.

### Műszaki szószedet

| Kifejezés | Meghatározás | Fő fájl |
| --- | --- | --- |
| LSN | Log Sequence Number (naplószámsor); a helyreállítás során használt WAL-rekordok egyedi azonosítója. |  |
| MPSC | Több termelő – egy fogyasztó; nagy teljesítményű szabad sorokhoz és GC-kötegelt feldolgozáshoz használják. |  139 |
| CRC32c | Ciklikus redundanciaellenőrzés (Castagnoli); az adatok integritásának biztosítására szolgál a kulcs-érték rekordokban és a WAL-ban. |  144 |
| Slab | Egy meghatározott méretű memóriablokk, amely azonos méretosztályba tartozó több elemet tartalmaz. |  |
| Piszkos oldal | Olyan memóriaoldal, amelyet már módosítottak, de még nem mentettek el az állandó tárolóba. |  |
| Arena | Gyors, régióalapú memóriatartószervező ideiglenes objektumokhoz (pl. lekérdezési tokenek). |  206 |
| SeqLock | Olyan szinkronizációs primitív, amely blokkolás nélkül teszi lehetővé a gyors, optimista olvasást. |  |
