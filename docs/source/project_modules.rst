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
            print(f"Could not request to APKMirror.\n{'You have been ratelimited' if req.status_code == 429 else f'Status code: {req.status_code}'}")
            return False