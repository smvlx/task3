Подготовка данных    
----------------------------------------

Используем данные, доступные по ссылке:  
<https://console.cloud.google.com/storage/browser/artem-pyanykh-cmc-prac-task3-seed17/out/input/>   

Для работы с Google Cloud используем пакет консольных инструментов gcloud. Устанавливаем gsutil как часть Google Cloud SDK.
Приведен алгоритм для Ubuntu. Для других ОС - <https://cloud.google.com/storage/docs/gsutil_install>

Работаем с коммандной строкой.

Настройки окружения для корректной загрузки:

    export CLOUD_SDK_REPO="cloud-sdk-$(lsb_release -c -s)"

Настройки источника пакета. Вы можете использовать 'https' вместо 'http' на этом шаге:

    echo "deb http://packages.cloud.google.com/apt $CLOUD_SDK_REPO main" | sudo tee -a /etc/apt/sources.list.d/google-cloud-sdk.list

Импортируем открытый ключ Google Cloud::

    curl https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
    
Обновляем и устанавливаем Cloud SDK:

    sudo apt-get update && sudo apt-get install google-cloud-sdk

Устанавливаем дополнительные пакеты для Python:

    sudo apt-get install google-cloud-sdk-app-engine-python
    
Если работаем в браузере, не загружающим автоматически URL, запускаем gcloud init, чтобы начать, так:

    gcloud init --console-only
   
Иначе:

    gcloud init
    
Видим сообщение:

>Welcome! This command will take you through the configuration of gcloud.   
>Your current configuration has been set to: [default] 
>To continue, you must login. Would you like to login (Y/n)?

Отвечаем "Y" для авторизации. Далее: 

>Your browser has been opened to visit:  
>https://accounts.google.com/o/oauth2/auth?redirect_uri=http%3A%2F%2F...

Если браузер не загружает автоматически URL:

>Go to the following link in your browser:  
>https://accounts.google.com/o/oauth2/auth?redirect_uri=urn%3Aietf%3Awg%3A...  
>Enter verification code:

Если окно браузера открылось автоматически, нажимаем "принять". 
После этого код подтверждения автоматически отправляется в командную строку.

Если работаем на удаленном компьютере или используем флаг --console only, копируем код подтверждения из URL-адреса
и вставляем его в командную строку терминала после

>Enter verification code:

Выбераем конфигурацию и проект по умолчанию для этой конфигурации.

После настройки учетных данных gcloud предложит проект по умолчанию для этой конфигурации и предоставит список доступных проектов. 
Выбираем идентификатор проекта из списка. Готово.

Теперь с помощью команды `cp` копируем файлы из облака на компьютер:

    gsutil cp gs://artem-pyanykh-cmc-prac-task3-seed17/out/input/*.csv .
    
Перемещаем все файлы в директорию под названием "init", также создаем директорию "answer", куда сохранятся наши ответы.
    
 Работа с данными
 -----------------------------
 Подключим некоторые библиотеки:
 
    import numpy as np
    import pandas as pd
    import glob as gb 
    import os
 
О структуре данных (предполагаемой): Четвертый символ в названии файла - штат. Пятый - номер магазина в штате. 

### Состояние склада на каждый день

Для каждого штата вызываем функцию `daily_store()`, которая возвращает массив DataFrame, длинна которого - число магазинов в штате. 

    dly = daily_store(files[j][shop][DAILY], files[j][shop][SPL])

**1. Прочитаем нужные файлы функцией `get_files(state, directory)`**

С помощью `path[shop] = gb.glob(directory + state + c + '\*.csv')` создадим массив имен нужных нам файлов, заданных маской.

В итоге получили массив прочитанных файлов `files[]`, где files\[shop] - так же является массивом: files\[shop]\[INV] - DataFrame, полученный из файла MS-{state shop}-inventory.csv  

Для каждого магазина:

**2. Преобразуем файл с кодами транзанкций проданных товаров функцией `sell_upg(sell)`**

Если первое вхождение подстроки 'ap' - 6ой символ, пишем в строке 0, иначе 1.

    upg_tmp['sku_num'] = sell['sku_num'].apply(lambda x: 0 if x.find('ap') == 6 else 1)
 
В новую таблицу метод `crosstab` запишет, используя 'date' как индекс, а '0' и '1' как столбцы, сколько значений '0' и '1' соответственно в каждом дне:
    
    daily = pd.crosstab(upg_tmp.index, upg_tmp['sku_num'], margins = True)
    
Для того, что привести получившиеся значения к одному типу, используем `values`

    daily_new['apple'] = daily[0].values
    daily_new['pen'] = daily[1].values
    
**3. Получаем файл-ответ**

Работаем с новым файлом продаж и данными о поставках. Итоговое количество товара на складе вычисляется как 'количество предыдущего дня' + 'поставка, если она была в этот день' - 'проданный товар'.

    df = pd.concat([files[shop][SPL], daily_new[])
    
Объединяем данные таблицы о поставках и "новые" данные о продажах, взятые с обратным знаком.

    df = pd.concat([files[shop][SPL], daily_new[]).sort_index()
    
Методом sort_index() сортируем значения по дате - получили таблицу о продажах, к значениям которой добавлены поставки, если они совершались в этот день.
 
###  Месячные данные о количестве сворованного товара ###

С помощью функции `files[j][shop][STOLEN] = stolen(files[j][shop][DAILY], files[j][shop][SPL], files[j][shop][INV])` получим DataFrame, содержащий данные о количестве сворованного товара.

**1. Получение месячных данных продаж и поступлений**

    sell = sell.resample('M').sum()
    supply = supply.resample('M').sum()

**2. Работа со вспомогательным DataFrame**

Создаем вспомогательный DataFrame `inv`, в котором хранятся данные об изменении инвентаря по сравнению с предыдущим месяцем, в первом месяце хранятся соответствующие данные без изменений. 
    
    inv['apple'] = inventory['apple'] - inventory['apple'] .shift(1)
    inv.loc[inv.index[0], 'apple'] = inventory.loc[inventory.index[0], 'apple']
    
**3. Подсчет количества сворованного товара**

Вычитаем из данных инвентаря за месяца суммарные поставки и добавляем суммарные продажи, получаем количество сворованного товара.
    
    st['apple'] = inv['apple'] + sell['apple'] - supply['apple']
    
    
    
###  Агрегированные данные об объемах продаж и количестве сворованной продукции по штату и году ###

**1. Создание массива из 3 DataFrame, по одному на каждый штат**

Каждый DataFrame создается с помощью вызова функции  `pd.DataFrame(index = sold[i].index)`, где `sold[i]` - DataFrame с информацией о проданном товаре за каждый рассматриваемый год.

**2. Добавление в DataFrame столбцов с информацией об украденном товаре**

В цикле в каждый DataFrame вставляются 3 новых столбца: apple_stolen, pen_stolen и state. 
    
    t[state].insert(2, 'apple_stolen', stolen[state]['apple'])
    t[state].insert(3, 'pen_stolen', stolen[state]['pen'])
    t[state].insert(0, 'state', states[state])
    
**3. Получаем конечную таблицу**

Объединяем DataFrame в один

    res = pd.concat([t[0], t[1], t[2]])
    

### Проверка результатов ### 

Для проверки результатов была реализована функция `check_res()`.
 
 
 
 
## Участники: 

1. Блохина Анна, 312 гр.
2. Нагаева Варвара, 312 гр.
3. Некрасова Мария, 312 гр.
4. Смирнов Александр, 312 гр.
