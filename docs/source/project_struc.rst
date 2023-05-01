Proje Yapısı
============

RVB PY'ın proje yapısı, `main.py` ana dosyasına ve `modules` klasörüne ayrılır:

::
    modules/
        ApkMirror/
                ApkFileDownloader.py
                Scraper.py
                __init__.py
        ADB.py
        Configuration.py
        FileDownloader.py
        GitHubAPI.py
        JavaChecker.py
        PatchRememberer.py
        PatcherProcess.py
        PatchesParser.py
    main.py
    cli.py

Mödüller
--------

ApkMirror Modülü
^^^^^^^^^^^^^^^^

ApkMirror modülü, apkmirror.com adlı siteden Android Paketlerini (APK) kazar (online scraping).

ApkFileDownloader modülü, APKMirror sitesinden seçilen uygulamayı indirir.

Scraper modülü ise seçilen uygulamanın versiyonlarını siteden kazar.

ADB Modülü
^^^^^^^^^^

ADB modülü bilgisayara bağlı olan cihazı kontrol etme, rootlu olup olmadığı kontrol etme,
ADB (Android Debugging Bridge) kurulu olup olmadığı kontrol etme ve cihazda yüklü olan uygulamanın
versiyonunu cihazdan alır.


Configuration Modülü
^^^^^^^^^^^^^^^^^^^^

Configuration modülü, bir RVB PY oturumundaki dosyaları, yamaları, önerilen uygulama sürümleri,
uygulama paketleri ve seçilen yamaları depolar. 

FileDownloader Modülü
^^^^^^^^^^^^^^^^^^^^^

FileDownloader modülü bir dosyayı indirmek için kullanılır.


GitHubAPI Modülü
^^^^^^^^^^^^^^^^

GitHubAPI modülü GitHub API'ını kullanarak GitHub'daki depoların (repositories) sürümlerini
alır ve indirir.

JavaChecker Modülü
^^^^^^^^^^^^^^^^^^

JavaChecker mödülü kullanıcının bilgisayarında Java yüklü olup olmadığını ve yüklü ise versiyonunu kontrol eder.

PatchRememberer Mödülü
^^^^^^^^^^^^^^^^^^^^^^

PatchRememberer mödülü seçtiğiniz yamaları kaydeder ve sonraki RVB PY kullanımınızda ise
seçtiğiniz yamaları yeniden seçer.

PatcherProcess Mödülü
^^^^^^^^^^^^^^^^^^^^^

PatcherProcess mödülü yamalama işlemini başlatır.

PatchesParser Mödülü
^^^^^^^^^^^^^^^^^^^^

PatchesParser mödülü indirilen yamaları ayrıştırır (parsing).