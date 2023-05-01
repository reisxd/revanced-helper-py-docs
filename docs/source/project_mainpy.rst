Ana Dosya, main.py
==================

RVB PY, modülleri kullanarak ana dosyada çalıştırır. Bu kısımda ana dosyanın 
ne yaptığı açıklanıcak.

Buradaki kod, RVB PY'ın gerektirdiği tüm modülleri yükler:

.. code-block:: python
    :linenos:

    import inquirer
    from modules.GitHubAPI import DownloadFiles
    from modules.PatchesParser import ParsePatches
    from modules.ApkMirror.Scraper import *
    from modules.ApkMirror.ApkFileDownloader import *
    from modules.PatcherProcess import RunPatcher
    from modules.ADB import CheckForRoot, GetPackageVersion
    from modules.JavaChecker import CheckJDKInstalled
    from modules.PatchRememberer import *
    import os
    import modules.Configuration


Buradaki kod, RVB PY'ın ayarlarını yükler:

.. code-block:: python
    :linenos:

    config = modules.Configuration.Configuration()
    selectedApp = {}

Buradan sonraki kodlar ise yorumlarla ne yaptığı belirtilecek:

.. code-block:: python
    :linenos:

    def main():

        # İlk öncelikle eğer 'revanced' klasörü yoksa onu oluşturur.
        if not os.path.exists('revanced'):
            os.mkdir('./revanced')

        # JDK/JRE yüklü değilse veya sürümü eski ise programdan çıkar
        if not CheckJDKInstalled():
            ExitApp()

        print('Welcome to ReVanced Builder PY! Please wait while Builder updates files.')

        # İlk öncelikle dosyaları indirip, sonra yamaları ayrıştırıp en son olarak paketleri
        # çeker (fetching)
        DownloadFiles(config)
        ParsePatches(None, config)
        apps = FetchPackages(config)
        appList = []
        for app in apps:
           appList.append((app['appName'], app))

        # Kullanıcıya 'inquirer' adlı TUI (Text-based User Interface) modülünü kullanarak
        # kullanıcıdan hangi uygulamayı seçmesini ister
        questions = [
            inquirer.List(
                "app",
                message="Please select the app you want to patch",
                choices=appList,
            ),
        ]

        answers = inquirer.prompt(questions)

        # Eğer cevap yok ise programdan çıkar.
        if answers == None:
            ExitApp()

        selectedApp = answers['app']

        # Seçilen uygulama için yamaları ayrıştırır
        ParsePatches(answers['app']['appPackage'], config)

        # Kullanıcıya önerilen (recommended) yamaları seçmesini mi ya da 
        # yamaları kendi seçip seçmeyeceğini sorar
        questions = [
            inquirer.List(
                "patches",
                message="Would you like to choose the patches yourself or choose the recommended patches",
                choices=[
                    ("Select recommended patches", True),
                    ("Select patches", False)
                ],
            ),
        ]

        answers = inquirer.prompt(questions)

        if answers == None:
            ExitApp()

        patchList = []

        # Eğer kullanıcı önerilen yamaları seçtiyse ise,
        # for döngüsü ile önerilen yamaları alıp bir listeye koyulur
        if answers['patches']:
            patches = config.GetPatches()['patches']
            for patch in config.GetPatches()['patches']:
               if patch['recommended']:
                   patchList.append(patch['name'])
               config.SetPatches('patches', patchList)
        else:
            # Eğer seçmediyse, tüm yamaları bir listeye koyulur
            selectedPatches = LoadPatches(selectedApp['appPackage'])
            for patch in config.GetPatches()['patches']:
                patchList.append(
                    (f"{patch['name']}\n   {patch['desc']}\n\n", patch['name']))

                # Kullanıcıdan istediği yamaların seçilmesi istenilir
                questions = [
                        inquirer.Checkbox(
                            "patches",
                            message="Please select the patches you want",
                            choices=patchList,
                            default=selectedPatches
                        ),
                ]

                answers = inquirer.prompt(questions)

                if answers == None:
                    ExitApp()

                # Kullanıcının seçtiği yamalar kaydedilir.
                WritePatches(selectedApp['appPackage'], answers['patches'])
                config.SetPatches('patches', answers['patches'])
                
                # Eğer seçilen uygulama YouTube ya da YouTube Music ise ve kullanıcı
                # microG Support yamasını seçmediyse, kullanıcının cihazına bir
                # Android cihaz takılı olup olmadığı kontrol edilir.
                # Edildikten sonra ise cihazın rootlu olup olmadığı kontrol edilir.
                # En son olarak ise cihazdaki uygulama sürümü alınır ve o sürüm indirilip
                # yamalama işlemi başlatılır.

                if (selectedApp['appPackage'] == 'com.google.android.youtube'
                    and 'microg-support' not in answers['patches']) or (selectedApp['appPackage'] == 'com.google.android.apps.youtube.music'
                                        and 'music-microg-support' not in answers['patches']):
        
                    deviceId = CheckForRoot()
                    if not deviceId:
                        ExitApp()
                    else:
                        DownloadAPK(
                            re.sub('\.', '-', GetPackageVersion(selectedApp), selectedApp))
                        RunPatcher(config, selectedApp)

        # Eğer seçilen uygulama zaten indirilmiş ise, kullanıcıya
        # bir daha kullanıp kullanmayacağı sorulur
        if os.path.exists(f"revanced/{selectedApp['appPackage']}.apk"):
             questions = [
                inquirer.Confirm(
                "downloadAPK",
                message="APK File already exists, do you want to download an another version"
            )
        ]

        answers = inquirer.prompt(questions)
        if answers == None:
            ExitApp()

        # Kullanmayacak ise yamalama işlemi başlatılır
        if not answers['downloadAPK']:
            RunPatcher(config, selectedApp)
            ExitApp()

        # Seçilen uygulama için sürümler çekilir
        versions = FetchVersions(selectedApp, config)

        versionList = []
        backslashChar = "\\"

        # Çekilen sürümler bir listeye koyulur
        for version in versions:
        versionList.append(
                (f"{re.sub(f'{backslashChar}-', '.', version['versionName'])} {'(Recommended)' if version['recommended'] else ''}", version))

        # Kullanıcıdan hangi sürümü kullanıcağı sorulur
        questions = [
            inquirer.List(
                "version",
                message="Please select the version you want to patch",
                choices=versionList,
            ),
        ]

        answers = inquirer.prompt(questions)

        if answers == None:
            ExitApp()

        # Seçilen uygulama ve sürüm indirilir
        DownloadAPK(answers['version']['versionName'], selectedApp)

        # Yamalama işletimi başlatılır
        RunPatcher(config, selectedApp)

        # En sonunda uygulamadan çıkılır.
        ExitApp()


    def ExitApp():
        input("Press any key to exit...")
        quit(0)