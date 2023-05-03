Modüller nasıl çalışır?
=======================

Bu bölümde modüllerin kodlarla birlikte nasıl çalıştığı anlatılacaktır.

ApkMirror modülleri
-------------------

ApkFileDownloader mödülü
^^^^^^^^^^^^^^^^^^^^^^^^

.. code-block:: python

    # Gereken modüller ilk öncelikle yüklenir
    import requests

    # BeautifulSoup modülü, web sayfalarını kazmak için kullanılır
    from bs4 import BeautifulSoup

    # Inquirer modülü, TUI (Text-based User Interface) modülüdür
    import inquirer

    # FileDownloader modülü, dosyaları indirmek için kullanılır
    from modules.FileDownloader import DownloadFile

    # MakeRequest fonksiyonu, request yaparken User-Agent header'ını eklemez.
    # Bunu ApkMirror websitesine requestlerken kullanılır.
    def MakeRequest(url):
        req = requests.get(url,
        headers={
            'User-Agent': None, 
            'Accept-Encoding': None
        })
        return req

    # DownloadAPK fonkisyonu, verilen sürümü ve uygulamayı indirir.
    def DownloadAPK(version, app):
        # İlk öncelikle verilen sürüm ile uygulamanın sayfasına request yapar.
        req = MakeRequest(f'https://www.apkmirror.com{app["link"]}{app["link"].split("/")[3]}-{version}-release/')
   
        # Eğer request'te bir problem yoksa, indirme işlemini başlat
        if req.status_code == 200:

            # BeautifulSoup modülünü kullanarak request'lenen sayfayı ayrıştırır (parse'lar).
            page = BeautifulSoup(req.content, "html.parser")

            # APK'nın universal (tüm cihazlarla uyumlu) olup olmadığını kontrol eder.
            universal = page.select("span[class='apkm-badge']")[0]
            notUniversal = page.find('div', class_='table-cell rowheight addseparator expand pad dowrap', string='arm64-v8a')

            # Değil ise, kullanıcıdan bir mimari (architecture) seçilmesi istenilir.
            if notUniversal:
                questions = [
                    inquirer.List(
                        "arch",
                        message="Please select the architecture you want to patch.\nYou can find this information on your devices settings or using CPU-Z",
                        choices=[
                            ("arm64-v8a"),
                           ("armeabi-v7a")
                        ],
                        ),
                    ]

                answers = inquirer.prompt(questions)

                apk = page.find('div', class_='table-cell rowheight addseparator expand pad dowrap', string=answers['arch'])
                apkLink = apk.find_previous_sibling('div').find('a')['href']
            else:
                apk = universal
                apkLink = apk.find_previous_sibling('a')['href']

            # Eğer APK linki yok ise, kullanıcıdan sürümü değiştirilmesi istenilir.
            if not apkLink:
                print(f'Could not find APK. Please try downgrading.')
                return False

            # Sonra ise APK linkine request'lenir ve sayfayı ayrıştırılır.
            req = MakeRequest(f'https://apkmirror.com{apkLink}')
            page = BeautifulSoup(req.content, "html.parser")

            # APK indirme linki alınır
            downloadLink = page.find('a', class_='accent_bg')['href']
        
            # APK indirme sayfasına request'lenir.
            req = MakeRequest(f'https://apkmirror.com{downloadLink}')
            page = BeautifulSoup(req.content, "html.parser")

            # APK indirme linki bulunur
            apkLink = page.find('a', attrs={ 'rel': 'nofollow' })['href']

            # En sonunda dosya indirilir.
            DownloadFile(f'revanced/{app["appPackage"]}.apk', f'https://apkmirror.com{apkLink}')
        else:
            # Eğer request'te bir problem varsa, hatanın ne olduğunu söyler.
            print(f"Could not request to APKMirror.\n{'You have been ratelimited' if req.status_code == 429 else f'Status code: {req.status_code}'}")
            return False

Scraper mödülü
^^^^^^^^^^^^^^

.. code-block:: python

    # Gerekilen modülleri yükle.
    import requests
    from bs4 import BeautifulSoup

    # re modülü, RegEx kullanmak içindir.
    import re


    # FetchPackages fonkisyonu, program ilk açıldığında yamalanabilir uygulamaları çeker.
    def FetchPackages(config):
        # İlk öncelikle bir API bitiş noktasına (endpoint) istek (request) atar.
        req = requests.post('https://www.apkmirror.com/wp-json/apkm/v1/app_exists/',
                            json={'pnames': config.GetPatches()['packages']},
                            headers={
                                'User-Agent': None,
                                'Accept-Encoding': None,
                                'Content-Type': 'application/json',
                                'Accept': 'application/json',
                                'Authorization':
                                'Basic YXBpLXRvb2xib3gtZm9yLWdvb2dsZS1wbGF5OkNiVVcgQVVMZyBNRVJXIHU4M3IgS0s0SCBEbmJL'
                            })

        apps = []
        # Eğer istekte bir problem yoksa, devam et
        if req.status_code == 200:
            # İstek cevabını bir dict'e çevir (JSON)
            res = req.json()
            # Tüm olan uygulamaları for döngüsü kullanarak bir listeye koy
            for app in res['data']:
                if app['exists']:
                    apps.append({
                        'appName': app['app']['name'].replace(' (Wear OS)', ''),
                        'appPackage': app['pname'],
                        'link': app['app']['link'].replace('-wear-os', '')
                    })
            return apps
        else:
            # İstekte bir problem varsa, kullanıcıya hatanın ne olduğu söyle
            print(
                f"Could not request to APKMirror.\n{'You have been ratelimited' if req.status_code == 429 else f'Status code: {req.status_code}'}")
            return False

    # FetchVersions fonksiyonu, bir uygulamanın APKMirror sayfasından sürümlerini kazar.
    def FetchVersions(app, config):
        versionList = []
        # Uygulama sayfasına istek atar.
        req = requests.get(f'https://www.apkmirror.com/uploads/?appcategory={app["link"].split("/")[3]}',
                           headers={
                               'User-Agent': None,
                               'Accept-Encoding': None
                           })
        # İstekte bir problem yoksa devam et.
        if req.status_code == 200:
            # Sayfa kazıma işlemine başla
            page = BeautifulSoup(req.content, "html.parser")
            primary = page.find(id="primary")
            versions = primary.find_all('h5')
            # Tüm sürümleri for döngüsüyle bir listeye koyar.
            for version in versions:
                if 'widgetHeader' in version['class']:
                    continue
                versionTitle = version['title'].lower()
                if 'download apkmirror' in versionTitle:
                        continue
                for child in version.contents:
                    # Eğer içerik sadece boşluk yada satır ise, bu "sürümü" geç
                    if child == '\n' or child == ' ':
                        continue
                    # RegEx kullanarak sürümü al
                    versionName = re.split(
                        f"(?<={app['link']}{app['link'].split('/')[3]}-)(.*)(?=-release/)", child['href'])[1]
                # Eğer sürüm isminde "release" yazısı yoksa, veya "(Wear OS)" yazısı varsa,
                # veya "-car_release" yazısı varsa, bu sürümleri geç.
                if (
                    app["appPackage"] == "com.twitter.android"
                    and not versionTitle.contains("release")
                ) or '(Wear OS)' in versionTitle or '-car_release' in versionTitle:
                    continue
                # En son olarak bir listeye sürümleri koy.
                versionList.append({
                    'versionName': versionName,
                    'recommended': re.sub("\-", ".", versionName) in config.GetPatches()['recommendedVersions'],
                    'beta': 'beta' in versionTitle
                })

            return versionList
        else:
            # İstekte bir problem var ise, kullanıcaya hatanın ne olduğunu söyle.
            print(
                f"Could not request to APKMirror.\n{'You have been ratelimited' if req.status_code == 429 else f'Status code: {req.status_code}'}")
            return False

ADB modülü
----------

.. code-block:: python

    # Gerekilen modülleri yükle.

    # subprocess modülü, yeni işlemleri başlatmak için kullanılır.
    import subprocess

    # re modülü, RegEx kullanmak içindir.
    import re

    # CheckADBInstalled fonksiyonu, kullanıcının bilgisayarında adb'nin yüklü olup olmadığını
    # kontrol eder.
    def CheckADBInstalled():
        try:
            # İlk öncelikle adb işlemini başlatır.
            subprocess.run(['adb'], stdout=subprocess.DEVNULL,
                           stderr=subprocess.DEVNULL)
        except FileNotFoundError:
            # Eğer bir hata döndürürse, kullanıcının bilgisayarında adb yüklü değildir.
            return False
        return True

    # GetFirstDevice fonksiyonu, kullanıcının bilgisayarına ilk takılı olan cihazı
    # çeker.
    def GetFirstDevice():
        # "adb devices" işlemini başlatır.
        result = subprocess.run(['adb', 'devices'], capture_output=True, text=True)
        
        # İşlem bittikten sonra, RegEx kullanarak bağlı cihaz olup olmadığını
        # kontrol eder.
        devices = re.search('\n(.*?)\t', result.stdout)
        if devices == None:
            return None
        # Eğer var ise, ilk olan cihazı çekip bazı şeyleri siler.
        return devices.group().replace('\n', '').replace('\t', '')

    # CheckRoot fonksiyonu, bilgisayara takılı olan cihazın rootlu olup olmadığını
    # kontrol eder.
    def CheckRoot():
        # "adb shell su -c exit" işlemini başlatır.
        result = subprocess.run(['adb', 'shell', 'su -c exit'],
                                capture_output=True, text=True)
        # Eğer cihaz bulunamadı hatası verirse, cihaz rootlu değildir.
        if 'inaccessible or not found' in result.stderr:
            return False
        return True

    # CheckForRoot fonksiyonu, hem adb'nin yüklü olmadığını, hem bir cihazın takılı olup olmadığını,
    # hem de cihazın rootlu olup olmadığı kontrol eder.
    def CheckForRoot():
        if not CheckADBInstalled():
            print('ADB is not installed. Please install it and connect your device.')
            return False
        else:
            deviceId = GetFirstDevice()
            if deviceId == None:
                print("There's no device plugged in. Please plug in a device.")
                return False
            if not CheckRoot():
                print("Your device is either not rooted or denied root access to shell.")
                return False
        return deviceId

    # GetPackageVersion fonksiyonu, cihazdaki uygulama sürümünü çeker.
    def GetPackageVersion(app):
        # "adb shell dumpsys package" işlemini başlatır.
        result = subprocess.run(['adb', 'shell', 'dumpsys', 'package',
                                app['appPackage']], capture_output=True, text=True)
        
        # RegEx kullanarak sürümü çeker.
        appVersion = re.split('versionName=([^=]+)', result.stdout)
        if appVersion == None:
            return None
        return appVersion[1].replace('\n    splits', '')

    # InstallMicroG fonksiyonu, kullanıcının cihazına Vanced microG yükler.
    def InstallMicroG(files):
        # "adb install revanced/VancedMicroG.apk" işlemini başlatır.
        subprocess.run(['adb', 'install', files['microg']],
                       stdout=subprocess.DEVNULL, stderr=subprocess.DEVNULL)

Configuration modülü
--------------------

.. code-block:: python

    class Configuration:
        def __init__(self):
            # Burada, indirilen dosyaların konumu yazılır.
            self.files = {
                'cli': '',
                'integrations': '',
                'patches': '',
                'patches-json': '',
                'microg': ''
            }

            # Burada, yamalar, seçilen uygulamanın önerilen sürümleri,
            # uygulama paketleri ve seçilen yamalar yazılır.
            self.patches = {
                'patches': [],
                'recommendedVersions': [],
                'packages': [],
                'includedPatches': []
            }

            # Buraya uygulamalar yazalır.
            self.apps = []

        # Uygulamaları verir.
        def GetApps(self):
            return self.apps
    
        # Uygulamaları yazar.
        def SetApps(self, val):
            self.apps = val

        # İndirilen dosyaları verir.
        def GetFiles(self):
            return self.files

        # İndirilen dosyaları yazar.
        def SetFiles(self, property, val):
            self.files[property] = val

        # Yamaları verir.
        def GetPatches(self):
            return self.patches

        # Yamaları yazar.
        def SetPatches(self, property, val):
            self.patches[property] = val

FileDownloader modülü
---------------------

.. code-block:: python

    # Gereken modüller yüklenir
    import requests

    # tqdm modülü, ilermeleme çubuğu (progress bar) için kullanılır.
    from tqdm import tqdm

    
    # DownloadFile fonksiyonu, dosya indirmek içindir.
    def DownloadFile(location, url):
        # İndirilecek dosyaya istek atılır.
        req = requests.get(url, stream=True,
                           headers={
                               'User-Agent': None,
                               'Accept-Encoding': None
                           })

        # Eğer iştekte bir problem yoksa, devam et.
        if req.status_code == 200:
            # Dosyanın büyüklüğünü çeker.
            total = int(req.headers.get('content-length', 0))
            # Dosyayı indirirken tdqm kullanılır.
            with open(location, 'wb') as file, tqdm(
                desc=location,
                total=total,
                unit='iB',
                unit_scale=True,
                unit_divisor=1024,
            ) as bar:
                for data in req.iter_content(chunk_size=1024):
                    size = file.write(data)
                    bar.update(size)

        else:
            # Eğer istekte bir problem varsa, kullanıcıya problemin ne olduğunu söyler.
            print(
                f"Could not request to {url}.\n{'You have been ratelimited' if req.status_code == 429 else f'Status code: {req.status_code}'}")
            return False

GitHubAPİ modülü
-----------------

.. code-block:: python

    # Gereken modüller yüklenir.
    import requests
    from modules.FileDownloader import DownloadFile
    import pathlib
    import os

    # FetchReleases fonksiyonu, GitHub API'ını kullanarak bir sürümü (release) çeker.
    def FetchReleases(repo):
        # GitHub API'ına istek atar.
        req = requests.get(f"https://api.github.com/repos/{repo['owner']}/{repo['repo']}/releases/latest")
        if req.status_code == 200:
            # İstek cevabını dict'e çevirir.
            res = req.json()
            
            # Varlıkları (assets) ve release sürümünü verir.
            return { 'assets': res['assets'], 'version': res['tag_name'] }
        else:
            print(f"Could not request to GitHub.\n{'You have been ratelimited' if req.status_code == 429 else f'Status code: {req.status_code}'}")
            return False

    # SetFiles fonksiyonu, indirilen dosyaları Configuration modülü kullanarak dosya konumlarını ayarlar.
    def SetFiles(repo, fileName, config):
        if repo['repo'] == 'revanced-cli':
            config.SetFiles('cli', fileName)
            return
        elif repo['repo'] == 'revanced-integrations':
            config.SetFiles('integrations', fileName)
            return
        elif repo['repo'] == 'revanced-patches' and fileName.endswith('.json'):
            config.SetFiles('patches-json', fileName)
            return
        elif repo['repo'] == 'revanced-patches':
            config.SetFiles('patches', fileName)
            return
        elif repo['repo'] == 'VancedMicroG':
            config.SetFiles('microg', fileName)
            return

    # DownloadFiles fonksiyonu, GitHub'dan dosyaları indirir.
    def DownloadFiles(config):
        # Dosyaları indirilicek olan GitHub depoları (repositories)
        repos = [
            {
                'owner': 'revanced',
                'repo': 'revanced-patches'
            },
            {
                'owner': 'revanced',
                'repo': 'revanced-integrations'
            },
            {
                'owner': 'revanced',
                'repo': 'revanced-cli'
            },
            {   
                'owner': 'TeamVanced',
                'repo': 'VancedMicroG'
            },
        ]

        for repo in repos:
            # İlk öncelikle bir sürümü çeker.
            assets = FetchReleases(repo)
            
            # Sonra, tüm varlıkları indirmeye başlar.
            for asset in assets['assets']:
                # Dosyanın uzantısını çeker,
                fileExt = pathlib.Path(asset['name']).suffix
                # Dosya adını belirler,
                fileName = f"revanced/{repo['repo']}-{assets['version']}{fileExt}"
                # Dosya konumunu ayarlar,
                SetFiles(repo, fileName, config)
                # `revanced` klasöründe olup olmadığını kontrol eder,
                revancedFolder = os.listdir('revanced')
                if fileName.split('/')[1] in revancedFolder:
                    continue
                # En son olarak dosyayı indirir.
                DownloadFile(fileName, asset['browser_download_url'])

JavaChecker modülü
------------------

.. code-block:: python

    # Gereken modülleri yükler
    import subprocess
    import re

    # https://stackoverflow.com/a/19859308

    # hasNumbers fonksiyonu, string'in içinde sayının olup olmadığını kontrol eder.
    def hasNumbers(inputString):
        return any(char.isdigit() for char in inputString)

    # CheckJDKInstalled fonksiyonu, Java/JDK'nin yüklü olup olmadığını ve
    # sürümünü kontrol eder.
    def CheckJDKInstalled():
        try:
            # "java -version" işlemini başlatır.
            result = subprocess.run(['java', '-version'], capture_output=True, text=True)
            javaLog = result.stdout or result.stderr

            # RegEx kullanarak parantezli olan tüm yazıları çeker.
            buildString = re.findall('\(.+?\)', javaLog)

            # Eğer hiç bir parantezli olan yazı yok ise, JDK/JRE yüklü değildir.
            if buildString == []:
                print('JDK is not installed.\nPlease install JDK from here: https://www.azul.com/downloads-new/?package=jdk')
                return False
            indx = 0

            # Yazılı olan parantezlerde sayı olup olmadığını kontrol eder.
            for i in range(0, len(buildString)):
                if hasNumbers(buildString[i]):
                    indx = i
                    break
            
            # Parantezin içindeki sürümü çeker.
            version = re.sub(r'[()]', '', buildString[indx].replace('build ', ''))
            versionNumbers = version.split('.')

            # Eğer sürüm eski ise, yeni bir sürüm yüklenmesi istenilir.
            if int(versionNumbers[0]) < 17 or 'openjdk' not in javaLog:
                print("JDK/Java was installed, but it's too old or not a JDK distribution.\nPlease install JDK from here: https://www.azul.com/downloads-new/?package=jdk")
                return False
            else:
                return True
        except FileNotFoundError:
            # Eğer yüklü değil ise, yüklemesi istenilir.
            print('JDK is not installed.\nPlease install JDK from here: https://www.azul.com/downloads-new/?package=jdk')
            return False

PatchRememberer modülü
----------------------

.. code-block:: python
    
    # Gereken modüller yüklenir.
    import json
    import os


    # CreateFile modülü, hatırlama dosyasını oluşturur.
    def CreateFile(value={'packages': []}):
        # Dosyayı yazar.
        f = open("config.json", "w")
        f.write(json.dumps(value))


    # WritePatches modülü, kullanıcının seçtiği yamaları hatırlar (yazar).
    def WritePatches(pkgName, patches):

        # İlk öncelikle dosyayı okur.
        f = open("config.json", "r+")
        configJson = json.load(f)
        found = False

        # Eğer paket bulunuyorsa, yamaları paketin içine yaz.
        for package in configJson['packages']:
            if package['name'] == pkgName:
                package['patches'] = patches
                found = True

        # Eğer bulunmuyorsa, paket listesine ekle.
        if not found:
            configJson['packages'].append({
                'name': pkgName,
                'patches': patches
            })

        # Dosyaya yaz.
        f.seek(0)
        json.dump(configJson, f)
        f.truncate()


    # LoadPatches fonksiyonu, hatırlanan yamaları yükler.
    def LoadPatches(pkgName):
        # Eğer dosya yoksa, dosyayı oluştur.
        if not os.path.exists('./config.json'):
            CreateFile()
            return []
        
        # Dosyayı okur.
        f = open("config.json", "r")
        configJson = json.load(f)
        # Eğer paket bulunursa, yamaları verir.
        for package in configJson['packages']:
            if package['name'] == pkgName:
                return package['patches']

        return []

PatcherProcess modülü
---------------------

.. code-block:: python

    # Gereken modüller yüklenir.
    import shutil
    import subprocess
    from modules.ADB import *

    # https://stackoverflow.com/a/72101287

    # RunCommand fonksiyonu, verilen komutları çalıştırır.
    def RunCommand(command, **kwargs):
        process = subprocess.Popen(
            command,
            stdout=subprocess.PIPE,
            stderr=subprocess.STDOUT,
            **kwargs,
        )
        while True:
            stdOut = process.stdout.readline()
            if not stdOut and process.poll() is not None:
                break
            print(stdOut.decode(), end='')

    # RunPatcher fonksiyonu, programın en son çalıştırdığı olan fonksiyondur.
    # Yamalama işlemini başlatır.
    def RunPatcher(config, app):
        # Gereken dosyaların konumlarını ve seçilen yamaları çeker.
        files = config.GetFiles()
        patches = config.GetPatches()
        hasDevice = False
        isRooted = False

        # Yamalayıcı başlatacak olan argümanlar.
        args = [
                'java',
                '-jar',
                files['cli'],
                '-b',
                files['patches'],
                '--experimental',
                '-a',
                f'revanced/{app["appPackage"]}.apk',
                '-o',
                f'revanced/ReVanced-{app["appName"]}.apk',
                '-m',
                files["integrations"],
                '--exclusive'
            ]
    
        # Eğer bir cihaz takılı ise, o cihaza yamalanmış uygulamayı yükle.
        if CheckADBInstalled():
            deviceId = GetFirstDevice()
            if deviceId:
                args.append('-d')
                args.append(deviceId)
                hasDevice = True
        # Eğer `microg-support` seçilin yamaların içinde değil ise, rootlu kurulumu başlat.
        if 'microg-support' not in patches['patches'] or (app['appPackage'] == 'com.google.android.apps.youtube.music' and 'music-microg-support' not in patches['patches']):
            args.append('--mount')
            isRooted = True

        # Seçilen yamaları ekle.
        for patch in patches['patches']:
            args.append('-i')
            args.append(patch)

        # Yamalayıcıyı başlat.
        RunCommand(args)
        shutil.rmtree('./revanced-cache')

        if hasDevice and isRooted:
            print('Successfully patched and mounted ReVanced!')
        elif hasDevice:
            # Eğer cihaz takılı, rootlu değil ve microg-support seçili ise Vanced microG'yide yükle.
            if not isRooted and 'microg-support' in patches['patches'] or 'music-microg-support' in patches['patches']:
                InstallMicroG(config.GetFiles())
            print('Successfully patched and installed ReVanced and installed Vanced MicroG!')
        else:
            # Eğer cihaz takılı değil ise, kullanıcıdan yamalanan uygulamayı cihazına
            # yüklemesini söyler.
            print(f"""Successfully patched ReVanced! Please transfer revanced/ReVanced-{app["appName"]}.apk
            and if you patched YT/YTM, also transfer revanced/{config.GetFiles()['microg']}.""")

PatchesParser modülü
--------------------

.. code-block:: python

    # Gereken modüller
    import json

    # ParsePatches fonksiyonu, indirilen yamaları ayrıştırır.
    def ParsePatches(packageName, config):
        # Yamalar dosyasını okur.
        with open(config.GetFiles()['patches-json']) as f:
            patches = json.load(f)
        for patch in patches:
            isCompatible = False
            
            # Yamanın paketle uyumlu olup olmadığınu kontrol eder.
            for package in patch['compatiblePackages']:

                # Program ilk açıldığında, uygulama paketlerini yükler.
                if package['name'] not in config.GetPatches()['packages']:
                    packages = config.GetPatches()['packages']
                    packages.append(package['name'])
                    config.SetPatches('packages', packages)
                
                # Eğer paket ismi yamanın uyumlu olduğu paketin ismine uymuyorsa
                # yamayı geçer.
                if package['name'] != packageName:
                    continue
                else:
                    # Uyumlu ise yamayı bir listeye koyar.
                    patches = config.GetPatches()['patches']
                    latestVer = package['versions'][-1] if package['versions'] else 'NONE'
                    patches.append({
                        'name': patch['name'],
                        'desc': patch['description'],
                        'latestVer': latestVer,
                        'recommended': not patch['excluded']
                    })
                    config.SetPatches('patches', patches)

                    # Tüm önerilen sürümleri çeker.
                    for version in package['versions']:
                        if version not in config.GetPatches()['recommendedVersions']:
                            versions = config.GetPatches()['recommendedVersions']
                            versions.append(version)
                            config.SetPatches('recommendedVersions', versions)