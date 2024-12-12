## 1. Сбор данных
<a id="data-collection"></a>
Сбор данных с трех веб-сайтов и сохранение в формате CSV.
  
  Используемые технологии: Python, Selenium, Beautiful Soup, Pandas, re.
  
  Сбор данных осуществляется четырьмя скриптами. Для примера выкладываю один. 

Комментарии и пояснения сделаны в коде

Сайт с бесконечной прокруткой. Нам нужно извлечь наименование и цену.

<details>
  <summary><strong>🖼️ Внешний вид сайта</strong></summary>

  ![Внешний вид сайта](https://raw.githubusercontent.com/sazhiromru/images/main/%D0%BC%D0%B0%D1%80%D0%BA%D0%B5%D1%82.PNG)
</details>

<details>
  <summary><strong>📜 Полный код скрипта</strong></summary>

```python
import pandas as pd
import undetected_chromedriver as uc
from bs4 import BeautifulSoup
from selenium.webdriver.chrome.service import Service
import re
import time
from selenium.webdriver.common.action_chains import ActionChains
import gc
from datetime import datetime
from random import uniform

timestamp = datetime.now().strftime('%d-%m-%Y')

def initialize_driver():
    ''' функция запуска Chrome со всеми возможными опциями для уменьшения нагрузки:
      - headless, минимальное разрешение, отключение загрузки картинок
      режим загрузки eager - по тестам так лучше грузится т к при загрузке идет скролинг'''
    options = uc.ChromeOptions()
    options.add_argument("--headless=new")
    options.add_argument("--disable-gpu")
    options.add_argument("--incognito")
    options.add_argument("--no-sandbox")
    options.add_argument("--disable-dev-shm-usage")
    options.add_argument("--window-size=800x600")
    options.add_argument("--disable-extensions")
    options.page_load_strategy = 'eager'
    options.experimental_options["prefs"] = {"profile.default_content_setting_values": {"images": 2}}
    
    service = Service(executable_path="chromedriver.exe")
    driver = uc.Chrome(service=service, options=options)
    driver.set_page_load_timeout(30)
    return driver


def extract_items(data):
    ''' функция для скрапа данных о предмете по шаблону наименование,цена'''
    soup = BeautifulSoup(data, 'html.parser')
    items_list = soup.find_all('div', class_=re.compile(r'item-info-block'))
    cleaned_list = [item.get_text().strip() for item in items_list if item.get_text().strip()]
    
    batch_data = []
    for item in cleaned_list:
        pattern = r'.+?\$(\d+\.\d+)\s-?\d+%\s(.+)'
        match = re.search(pattern, item)
        if match:
            batch_data.append([match.group(2).replace(',', ''), match.group(1).replace(',', '')])
    return batch_data


def scroll_and_extract(driver, duration, output_file):
    '''Создаем файл с BOM отдельно - несколько раз были ошибки, это помогло
    Т к китайские йероглифы и множество спец знаков в наименованиях сохраняем в utf-16
    загрузка и очистка данных пачками по 2200, число подобрано из практики, баланс между памятью и I/O
    Так же есть несколько условий отлова ошибок и перезапуска драйвера
    Ключ к успешной работе - периодическая очистка кэша'''

    start_time = time.time()
    batch_data = []
    global timestamp

    with open(f'{output_file}_{timestamp}.csv', 'wb') as file:
        file.write(b'\xff\xfe')
        pd.DataFrame(columns=["Item", "market_price", "Date"]).to_csv(file, index = False, encoding = 'utf-16')

    while True:
        try:
            data = driver.page_source
            batch_data.extend(extract_items(data))
            
            ActionChains(driver).scroll_by_amount(0, 600).perform()
            time.sleep(uniform(3.44, 5.17))

            if len(batch_data) >= 2200:
                df = pd.DataFrame(batch_data, columns = ['Item','market_price'])
                df = df.drop_duplicates()
                df.to_csv(f'{output_file}_{timestamp}.csv', mode = 'a', index=False, header = False, encoding = 'utf-16')
                batch_data.clear()
                df = None
                driver.execute_cdp_cmd("Network.clearBrowserCache", {})
                gc.collect()

            if time.time() - start_time > duration:
                print("время вышло.")
                break
        except Exception as e:
            print(f"ошибка промотка: {e}")
            driver.quit()
            time.sleep(22)
            driver = initialize_driver()
            time.sleep(22)


    if batch_data:
        df = pd.DataFrame(batch_data, columns = ['Item','market_price'])
        df = df.drop_duplicates()
        df.to_csv(f'{output_file}_{timestamp}.csv', mode = 'a', index=False, header = False, encoding = 'utf-16')



def cleaning(file_name):
    global timestamp
    """финальные округления, удаление пробелов и проверка на дубликаты"""
    df = pd.read_csv(file_name, encoding='utf-16')
    df['market_price'] = df['market_price'].astype(float).round(4)
    df['Item'] = df['Item'].str.strip()
    df['Date'] = timestamp
    df = df.sort_values(by='Item').drop_duplicates(subset=['Item']).reset_index(drop=True)
    df.to_csv(file_name, index=False, encoding='utf-16')
    print(f"Дата загружена в {file_name}")


def main():
    timestamp = datetime.now().strftime('%d-%m-%Y')
    output_file = 'market'
    duration = 1322
    
    driver = initialize_driver()
    print('драйвер запущен')
    
    driver.get('https://market.csgo.com/en/?priceMin=4')
    time.sleep(5)  
    
    scroll_and_extract(driver,duration = duration, output_file = output_file)
    print('Промотка окончена.')
    
    driver.quit()
    print('Драйвер закрыт.')
    
    cleaning(f'{output_file}_{timestamp}.csv')
    print('Очистка закончена.')


if __name__ == "__main__":
    main()
```

</details>
<br></br>

## 2. Обработка данных
<a id="data-wrangling"></a>

### Merge.
Внешним слиянием соединяем три csv, округялем цифры, приводим валюту к доллару, убираем дубликаты, ошибки и находим самые выгодные сделки по продаже китай->рф и рф->китай
<details>
  <summary><strong>📜 Merge код</strong></summary>

```python
import pandas as pd
import numpy as np
from datetime import datetime

timestamp = datetime.now().strftime('%d-%m-%Y')

path_c5 = f'c5game_{timestamp}.csv'
path_market = f'market_{timestamp}.csv'
path_buff = f'buff_{timestamp}.csv'
path_buff_buyorders = f'buff_buyorders_{timestamp}.csv'

#автоматического обновления курса нет, но курс существенно юань/доллар не меняется много лет. 
cny_usd = 0.14
profit_coef = 0.9025

df_c5 = pd.read_csv(path_c5, encoding = 'utf-16')
df_c5.drop_duplicates()
df_c5.rename(columns = {'c5_item':'Item'},inplace = True)
#Создан фрейм с5 с колонками 'Item', 'c5_price'

df_buff_buyorders = pd.read_csv(path_buff_buyorders, encoding = 'utf-16')
df_buff_buyorders.rename(columns = {'buff_item':'Item', 'buff_price':'price'}, inplace = True)
#Создан фрейм buff_buyorders с колонками 'Item', 'buyorders_price'

df_market = pd.read_csv(path_market, encoding = 'utf-16')
#Создан фрейм market с колонками 'Item', 'market_price'

df_buff = pd.read_csv(path_buff, encoding = 'utf-16')
df_buff.rename(columns = {'buff_item':'Item'},inplace = True )
#Создан фрейм buff с колонками 'Item', 'buff_price'

df_final = pd.merge(df_buff, df_market, on = 'Item', how = 'outer')
df_final = pd.merge(df_final, df_c5, on = 'Item', how = 'outer')
# внешний мердж

df_final['c5_price'] =  df_final['c5_price'].astype(float).fillna(0)
df_final['buff_price'] =  df_final['buff_price'].astype(float).fillna(0)
df_final['market_price'] =  df_final['market_price'].astype(float).fillna(0)

df_final['buff_price'] = df_final['buff_price'].astype(float).apply(lambda x : x*cny_usd)
df_final['c5_price'] = df_final['c5_price'].astype(float).apply(lambda x : x*cny_usd)
# выполнены округление и очистка, приведение йен к долларам

'''Сравнение цен по продаже китай -> маркет, выбор лучших сделок на c5game and buff163'''
df_direct = df_final.copy(deep = True)

df_direct['market_price'] = df_direct['market_price'].astype(float).apply(lambda x: x*profit_coef)
df_direct['market_c5'] = (df_direct['market_price']/df_direct['c5_price'])
df_direct['market_buff'] = (df_direct['market_price']/df_direct['buff_price'])
df_direct.replace([np.inf, -np.inf], 0, inplace=True)

df_direct['best_price'] = np.maximum(df_direct['market_buff'], df_direct['market_c5'])
df_direct['label'] = np.where(df_direct['market_buff']>df_direct['market_c5'],'buff','c5')
df_direct = df_direct.sort_values(by = 'best_price', ascending=False)
#

columns_to_round = ['c5_price', 'market_price', 'buff_price', 'market_c5', 'market_buff', 'best_price']
df_direct[columns_to_round] = df_direct[columns_to_round].round(4)
df_direct['Item'] = df_direct['Item'].str.strip()
df_direct.drop_duplicates(subset=['Item'], inplace=True)
df_direct.reset_index(drop=True, inplace=True)

df_direct.drop(columns=['Date','Date_x','Date_y'], inplace=True)
df_direct['Date'] = timestamp

df_direct.drop_duplicates(inplace=True)

df_direct.to_csv(f'direct_{timestamp}.csv',index = False, encoding = 'utf-16')

'''Сравнение цен по продаже маркет -> китай, выбор лучших сделок по ордерам на покупку на бафф'''
df_reverse = pd.merge(df_market, df_buff_buyorders, on = 'Item', how = 'outer')
df_reverse = df_reverse.fillna(0)
df_reverse['buyorders_price'] = df_reverse['buyorders_price'].astype(float).apply(lambda x: x*cny_usd)
df_reverse['coef'] = df_reverse['buyorders_price'] / df_reverse['market_price']
df_reverse.replace(np.inf,0, inplace = True)
df_reverse.sort_values(by = 'coef', ascending=False, inplace = True)

df_reverse.drop(columns=['Date_x','Date_x'], inplace=True)
df_reverse['Date'] = timestamp
df_reverse.drop_duplicates(inplace = True)

df_reverse.to_csv(f'reverse_{timestamp}.csv',index = False, encoding = 'utf-16')
```
</details>  


### Создание ссылок для проверки рекомендаций.
120 предметов с потенциально лучшей прибылью проверяются по истории продаж за последний месяц. Для этого необходимо сгенерировать ссылку для перехода на страницу предмета на сайта market-csgo.
В соответствии со структурой данных на сайте создан справочник, позволяющий сконструировать ссылку по названию предмета.

Из интересного - символы '™' и '★'. Для генерации категорий и корректным сравнением со справочником символы удаляются специальной функцией. Для генерации ссылки символы меняются на специальные плейсхолдеры и обратно, для результата в виде AK-47/StatTrak™%20AK-47 или Shadow%20Daggers/★%20Shadow%20Daggers
<details>
  <summary><strong>📜 Url_creator код</strong></summary>
  
```python
import pandas as pd
import re
from urllib.parse import quote
from datetime import datetime

timestamp = datetime.now().strftime('%d-%m-%Y')
path = f'direct_{timestamp}.csv'

df = pd.read_csv(path,encoding = 'utf-16',on_bad_lines='skip')

def categorization(item):
    category_mapping = {
        "Knife": [
            "Bayonet", "Bowie Knife", "Butterfly Knife", "Classic Knife", "Falchion Knife", "Flip Knife", "Gut Knife", 
            "Huntsman Knife", "Karambit", "M9 Bayonet", "Navaja Knife", "Nomad Knife", "Paracord Knife", "Skeleton Knife", 
            "Stiletto Knife", "Survival Knife", "Talon Knife", "Ursus Knife", "Kukri Knife", "Shadow Daggers",
            "Butterfly Knife | Fade"
        ],
        "Agent": [
            "Agent", "Master Agent", "Exceptional Agent", "Superior Agent", "Sir Bloody", "Lt. Commander", "Vypa Sista",
            "Buckshot", "Special Agent", "Chem-Haz Capitaine", "Dragomir", "Cmdr. Davida", "Slingshot", "Chef d'Escadron",
            "Safecracker", "1st Lieutenant", "Number K", "Enforcer", "Markus Delrow", "Maximus", "Ricksaw", "Goggles",
            "Crasswater The Forgotten", "Trapper Aggressor", "Lieutenant Rex Krikey", "Col. Mangos Dabisi",
            "B Squadron Officer", "Trapper", "Cmdr. Frank 'Wet Sox' Baroud", "John 'Van Healen' Kask", "Little Kev",
            "D Squadron Officer", "Bio-Haz Specialist", "Chem-Haz Specialist", "'Medium Rare' Crasswater",
            "Operator", "The Elite Mr. Muhlik", "Street Soldier", "Getaway Sally", "Bloody Darryl The Strapped",
            "3rd Commando Company", "Cmdr. Mae 'Dead Cold' Jamison", "Sous-Lieutenant Medic", "Michael Syfers",
            "Jungle Rebel", "Elite Trapper Solman", "Sergeant Bombson", "Officer Jacques Beltram", "Arno The Overgrown",
            "'The Doctor' Romanov", "Osiris", "'Two Times' McCoy", "Blackwolf", "Aspirant",
            "Lieutenant 'Tree Hugger' Farlow", "Ground Rebel", "Primeiro Tenente", "Prof. Shahmat", 
            "Rezan The Ready", "Rezan the Redshirt", "Seal Team 6 Soldier"
        ],
        "Pistol": ["USP-S", "Glock-18", "P2000", "Desert Eagle", "Five-SeveN", "CZ75-Auto", "P250", "Dual Berettas", "Tec-9", "R8 Revolver"],
        "Rifle": ["AK-47", "M4A4", "M4A1-S", "FAMAS", "Galil AR", "SG 553", "AUG"],
        "Sniper Rifle": ["AWP", "SSG 08", "SCAR-20", "G3SG1"],
        "SMG": ["MP7", "P90", "UMP-45", "MAC-10", "MP9", "PP-Bizon", "MP5-SD"],
        "Shotgun": ["Nova", "XM1014", "MAG-7", "Sawed-Off"],
        "Machine Gun": ["M249", "Negev"],
        "Gloves": [
            "Gloves", "Hand Wraps", "Driver Gloves", "Moto Gloves", "Specialist Gloves", "Sport Gloves",
            "★ Hand Wraps | Spruce DDPAT (Field-Tested)"
        ],
        "Container": [
            "Case", "Capsule", "Container", 
            "Music Kit | Feed Me, High Noon",
            "StatTrak™ Initiators Music Kit Box", 
            "StatTrak™ NIGHTMODE Music Kit Box", 
            "StatTrak™ Masterminds 2 Music Kit Box", 
            "StatTrak™ Masterminds Music Kit Box", 
            "StatTrak™ Tacticians Music Kit Box",
            "Berlin 2019 Vertigo Souvenir Package", 
            "Rio 2022 Vertigo Souvenir Package",
            "Stockholm 2021 Vertigo Souvenir Package",
            "ESL One Cologne 2014 Challengers",
            "ESL One Cologne 2014 Legends",
            "Katowice 2019 Legends (Holo/Foil)",
            "Katowice 2019 Minor Challengers (Holo/Foil)",
            "Gift Package"
        ],
        "Music Kit": ["Music Kit"],
        "Sticker": ["Sticker"],
        "Charm": ["Charm"],
        "Graffiti": ["Graffiti"],
        "Patch": ["Patch"],
        "Pass": ["Pass"],
        "Equipment": ["Zeus"],
        "Collectible": ["Pin"],
        "Key": ["Key", "eSports Key"] 
    }

    for key, values in category_mapping.items():
        for value in values:
            if value.lower() in item.lower():
                return key
    return "Unknown"

def subcategory(item):
    subcategory_mapping = {
        "Pistol": ["USP-S", "Glock-18", "P2000", "Desert Eagle", "Five-SeveN", "CZ75-Auto", "P250", "Dual Berettas", "Tec-9", "R8 Revolver"],
        "Rifle": ["AK-47", "M4A4", "M4A1-S", "FAMAS", "Galil AR", "SG 553", "AUG"],
        "Sniper Rifle": ["AWP", "SSG 08", "SCAR-20", "G3SG1"],
        "SMG": ["MP7", "P90", "UMP-45", "MAC-10", "MP9", "PP-Bizon", "MP5-SD"],
        "Shotgun": ["Nova", "XM1014", "MAG-7", "Sawed-Off"],
        "Machine Gun": ["M249", "Negev"]}
    for key, values in subcategory_mapping.items():
        for value in values:
            if value in item:
                return value
    return None

def normalization(item):
    item = re.sub('★','',item)
    item = re.sub('StatTrak™','',item)
    return item
    
df['Item_cleaned'] = df['Item'].apply(lambda x: normalization(x))
df['Category'] = df['Item_cleaned'].apply(lambda x: categorization(x))
df['Subcategory'] = df['Item_cleaned'].apply(lambda x: subcategory(x))

base = 'https://market.csgo.com/en/'
def custom_quote(item):
    
    item = item.replace('™', 'PLACEHOLDER_TM').replace('★', 'PLACEHOLDER_STAR')
    
    encoded = quote(item)
   
    return encoded.replace('PLACEHOLDER_TM', '™').replace('PLACEHOLDER_STAR', '★')

df['url'] = df.apply(lambda row: f"{base}{row['Category']+'/'}"f"{row['Subcategory']+'/' if pd.notna(row['Subcategory']) else ''}"f"{custom_quote(row['Item'])}", axis=1)
df.to_csv(path, encoding = 'utf-16', index = False)
```
</details>  


### Проверка 120 предметов по созданным ссылкам
На данном этапе наша задача извлечь со 120 страниц данные по которым строится график продаж предмета, цену запроса на покупку, и уточнить цену продажи.  
Для последующей обработки считается средняя цена сделок за последние 4 суток, и количество этих сделок.
<details>
  <summary><strong>🖼️ Страница предмета</strong></summary>

  ![Внешний вид сайта](https://raw.githubusercontent.com/sazhiromru/images/main/item%20page.PNG)
</details>
<details>
  <summary><strong>📜 Полный код скрипта</strong></summary>

```python
import pandas as pd
import undetected_chromedriver as uc
from bs4 import BeautifulSoup
from selenium.webdriver.chrome.service import Service
import re
import time
from datetime import datetime
from random import uniform

timestamp = datetime.now().strftime('%d-%m-%Y')

def get_medium_price(history):
    now = datetime.now()
    list1=[]
    for record in history:
        item1 = [item.strip() for item in record.split(',')]
        item1.pop(2)
        item1.pop(0)
        item1[1] = item1[1].replace('. Price.','')
        list1.append(item1)
        date_format = "%b %d"
    
    print(list1)
    filtered_list = [
        item for item in list1
        if (now - datetime.strptime(item[0], date_format).replace(year=2024)).days < 4
    ]
    print(filtered_list)    
    sum1=0
    if len(filtered_list)<=2:
        history_price = 'меньше двух сделок за 4 дня'
    else:
        for price2 in filtered_list:
            sum1 += float(price2[1])
            history_price = sum1/len(filtered_list)
    print(history_price)
    return history_price

def frequency_calc(history):
    now = datetime.now()
    list1=[]
    for record in history:
        item1 = [item.strip() for item in record.split(',')]
        item1.pop(2)
        item1.pop(0)
        item1[1] = item1[1].replace('. Price.','')
        list1.append(item1)
        date_format = "%b %d"
    filtered_list = [
        item for item in list1
        if (now - datetime.strptime(item[0], date_format).replace(year=2024)).days < 4
    ]

    if len(filtered_list)<=2:
        frequency = 'low'
    elif len(filtered_list)<=5:
        frequency = 'medium'
    elif len(filtered_list)<=9:
        frequency = 'high'
    else:
        frequency = 'very high'
    return frequency


def initialize_driver():

    options = uc.ChromeOptions()
    options.add_argument("--headless=new")  
    options.add_argument("--disable-gpu")  
    my_user_agent = "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/108.0.0.0 Safari/537.36"
    options.add_argument("--incognito") 
    options.add_argument(f"--user-agent={my_user_agent}")
    options.add_argument('--disable-extensions')
    options.add_argument("--disable-plugins-discovery")
    options.add_argument("--no-sandbox")  
    options.add_argument("--disable-dev-shm-usage")  
    options.add_argument("--window-size=800x600")  
    options.add_argument("--disable-extensions")  
    options.add_argument("--disable-software-rasterizer") 
    
    chrome_prefs = {
        "profile.default_content_setting_values": {
            "images": 2,
        }
        }
    
    options.page_load_strategy = 'eager'
    options.experimental_options["prefs"] = chrome_prefs

    service = Service(executable_path="chromedriver.exe")
    driver = uc.Chrome(service=service, options=options)
    driver.set_page_load_timeout(60) 
    return driver

path = f'direct_{timestamp}.csv'
number = 100
k=0
df = pd.read_csv(path, encoding = 'utf-16')
driver = initialize_driver()


bad_url=[]
history_prices = []
actual_prices = []
request_prices = []
frequency_list = []
soup = None
duration = 2000

for url in df['url']:
    start = time.time()
    attempts = 0
    while attempts<=2:
        try:
            driver.get(url)
            time.sleep(uniform(5.45,7.11))
            data = driver.page_source
            soup = BeautifulSoup(data,'html.parser')
            prices = soup.find_all('div', class_='price')
            prices1 = [price.get_text() for price in prices]
            if prices!=[]:
                break
        except Exception as e:
            driver.quit()
            time.sleep(10)
            driver = initialize_driver()
            time.sleep(10)
            print(f'возниклда ошибка {e}')
            attempts+=1

    if any('₽' in price for price in prices1):
        history_prices.append('ошибочный url')
        request_prices.append('ошибочный url')
        actual_prices.append('ошибочный url')
        frequency_list.append('ошибочный url')
        k+=1
        continue


    if prices1[3].startswith(' '):
        actual_prices.append('нет цены на покупку')
        request_prices.append(prices1[3])
    else:
        prices1[-1] = prices1[-1].replace('≤ ', '').replace('$', '').replace(' ','')
        request_prices.append(prices1[-1])

        pattern = r'\$(\d+(\.\d+)?)'
        match = re.search( pattern, prices1[3])
        if match:
            actual_prices.append(match.group(1))
            print(match.group(1))
        else:
            actual_prices.append('цена отсутствует')
            print('цена отсутсвует')


    history_charts = soup.select('path.highcharts-point.highcharts-color-0')
    history = [chart.get('aria-label') for chart in history_charts]
    history_prices.append(get_medium_price(history))
    frequency_list.append(frequency_calc(history))

   
    print(f'страница номер {k} обработана')
    print(len(actual_prices))
    print(len(request_prices))
    print(len(history_prices))
    k+=1

    if k%10==0:
        time.sleep(5)
        print('перегрузка')
        driver.execute_cdp_cmd("Network.clearBrowserCache", {})
        time.sleep(5)

    if k>number:
        print('конец')
        driver.quit()
        print(len(actual_prices))
        print(len(request_prices))
        print(len(history_prices))
        break
        
    if (time.time() - start) > duration:
        driver.quit()
        break

df['Medium Prices'] = None
df['Actual Prices'] = None
df['Request price'] = None
df['Frequency'] = None

df.loc[0:number, 'Medium Price'] = history_prices
df.loc[0:number, 'Actual Prices'] = actual_prices
df.loc[0:number, 'Request price'] = request_prices
df.loc[0:number, 'Frequency'] = frequency_list
print('данные загружены')

if bad_url!=[]:
    df_badurl = pd.DataFrame(bad_url)
    df_badurl.to_csv('badurl.csv')

df.to_csv(path, encoding='utf-16',index = False)
print('файл сохранен')
```
</details>  

### Ранжирование
По полученным данным, и произвольной эмпирической формуле ранжируем сделки по степени привлекательности.  
Так же, проверяем все сделки на прибыльность на случай ошибки алгоритмов/скрапинга
<details>
  <summary><strong>📜 Полный код скрипта</strong></summary>

```python
import pandas as pd
from datetime import datetime

timestamp = datetime.now().strftime('%d-%m-%Y')

path = f'direct_{timestamp}.csv'

df = pd.read_csv(path, encoding = 'utf-16')
columns_con = ['Request price','buff_price','c5_price','best_price', 'Actual Prices', 'market_price', 'Medium Price']
df[columns_con] = df[columns_con].apply(pd.to_numeric, errors = 'coerce')

def calculate_rating_coef(row):
    valid_min = min([value for value in [row['buff_price'], row['c5_price']] if value > 0], default=1)
    
    if row['Frequency'] == 'low' and row['Request price'] < valid_min:
        return 0
    elif row['Frequency'] == 'medium':
        return 1.2 * ((row['Medium Price'] / valid_min)**3) * (valid_min**0.16)
    elif row['Frequency'] == 'high':
        return 1.4 * ((row['Medium Price'] / valid_min)**3) * (valid_min**0.16)
    elif row['Frequency'] == 'very high':
        return 1.6 * ((row['Medium Price'] / valid_min)**3) * (valid_min**0.16)
    else:
        return 1 
    
def check_profit(row):
    valid_min = min([value for value in [row['buff_price'], row['c5_price']] if value > 0], default=1)
    if row['Medium Price'] * 0.9 < valid_min*1.03:
        return 0
    else:
        return 1
 

df['Rating'] = df.apply(calculate_rating_coef, axis=1)
df['Check'] = df.apply(check_profit, axis = 1)  
df['Rating'] = df['Rating'] * df['Check']

df = df.sort_values(by='Rating', ascending=False)
print(df.head())
df.to_csv(path, encoding= 'utf-16', index=False)
```
</details>  
<br></br>  

## 3. SQL
<a id="SQL"></a>

### Создание БД
Для управления/просмотра данных используем PgAdmin. Подключение через SHH public ip ec2, т е bastion server.  
<details>
  <summary><strong>🖼️ SSH подключение к серверу</strong></summary>

  ![Connection](https://raw.githubusercontent.com/sazhiromru/images/main/ssh%20SQL%20bastion%20server.PNG)
  ![SSH tunnel](https://raw.githubusercontent.com/sazhiromru/images/main/sql%20SSH%20tunnel.PNG)
</details>

Под 4 скрапинга создаем 4 таблицы. Так же для продаж Китай - РФ и РФ - Китай по одной таблице.
Уникальным ключом каждой таблицы является дата+название предмета  
<details>
  <summary><strong>🖼️ PGAdmin</strong></summary>

  ![Внешний вид сайта](https://raw.githubusercontent.com/sazhiromru/images/main/sql.PNG)
</details>  

### Загрузка данных в таблицы 
При загрузке прописываем конфликты по ключу, т к при скрапинге невозможно получить идеальные данные  

<details>
  <summary><strong>📜 Полный код скрипта</strong></summary>

```python
from datetime import datetime
import os
import pandas as pd
import psycopg2

timestamp = datetime.now().strftime('%d-%m-%Y')

conn = psycopg2.connect(
    host=
    database=
    user=
    password=
)
cursor = conn.cursor()

try:
    for filename in os.listdir(os.path.dirname(os.path.abspath(__file__))):
        if filename == f'market_{timestamp}.csv':
            df = pd.read_csv(filename, encoding='utf-16')
            df['Date'] = pd.to_datetime(df['Date'], format='%d-%m-%Y').dt.strftime('%Y-%m-%d')
            for index, row in df.iterrows():
                cursor.execute(
                    """INSERT INTO public.market (item, price, date, id) 
                    VALUES (%s, %s, %s, %s)
                    on conflict(id)
                    do update set
                    item = excluded.item,
                    price = excluded.price,
                    date = excluded.date""",
                    (row['Item'], row['market_price'], row['Date'], row['Item'] + ' ' + str(row['Date']))
                )
    conn.commit()
    print('market has been loaded')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()

try:
    for filename in os.listdir(os.path.dirname(os.path.abspath(__file__))):
        if filename == f'buff_{timestamp}.csv':
            df = pd.read_csv(filename, encoding='utf-16')
            df['Date'] = pd.to_datetime(df['Date'], format='%d-%m-%Y').dt.strftime('%Y-%m-%d')
            for index, row in df.iterrows():
                cursor.execute(
                    """INSERT INTO public.buff_sell (item, price, date, id) 
                    VALUES (%s, %s, %s, %s)
                    on conflict(id)
                    do update set
                    item = excluded.item,
                    price = excluded.price,
                    date = excluded.date""",
                    (row['Item'], row['buff_price'], row['Date'], row['Item'] + ' ' + str(row['Date']))
                )
    conn.commit()
    print('buff has been loaded')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()

try:
    for filename in os.listdir(os.path.dirname(os.path.abspath(__file__))):
        if filename == f'buff_buyorders_{timestamp}.csv':
            df = pd.read_csv(filename, encoding='utf-16')
            df['Date'] = pd.to_datetime(df['Date'], format='%d-%m-%Y').dt.strftime('%Y-%m-%d')
            for index, row in df.iterrows():
                cursor.execute(
                    """INSERT INTO public.buff_buyorders (item, price, date, id) 
                    VALUES (%s, %s, %s, %s)
                    on conflict(id)
                    do update set
                    item = excluded.item,
                    price = excluded.price,
                    date = excluded.date""",
                    (row['Item'], row['buyorders_price'], row['Date'], row['Item'] + ' ' + str(row['Date']))
                )
    conn.commit()
    print('buff buy has been loaded')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()

try:
    for filename in os.listdir(os.path.dirname(os.path.abspath(__file__))):
        if filename == f'c5game_{timestamp}.csv':
            df = pd.read_csv(filename, encoding='utf-16')
            df['Date'] = pd.to_datetime(df['Date'], format='%d-%m-%Y').dt.strftime('%Y-%m-%d')
            for index, row in df.iterrows():
                cursor.execute(
                    """INSERT INTO public.c5 (item, price, date, id) 
                    VALUES (%s, %s, %s, %s)
                    on conflict(id)
                    do update set
                    item = excluded.item,
                    price = excluded.price,
                    date = excluded.date""",
                    (row['Item'], row['c5_price'], row['Date'], row['Item'] + ' ' + str(row['Date']))
                )
    conn.commit()
    print('c5 has been loaded')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()

try:
    for filename in os.listdir(os.path.dirname(os.path.abspath(__file__))):
        if filename == f'direct_{timestamp}.csv':
            df = pd.read_csv(filename, encoding='utf-16')
            df['Date'] = pd.to_datetime(df['Date'], format='%d-%m-%Y').dt.strftime('%Y-%m-%d')
            for index, row in df.iterrows():
                cursor.execute(
                    """INSERT INTO public.direct (item, buff_price, market_price, c5_price, market_c5, market_buff, best_price, label, date, category, frequency, medium_price, rating, id)
                    VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
                    on conflict(id)
                    do update set
                    item = excluded.item,
                    buff_price = excluded.buff_price,
                    market_price = excluded.market_price,
                    c5_price = excluded.c5_price,
                    market_c5 = excluded.market_c5,
                    market_buff = excluded.market_buff,
                    best_price = excluded.best_price,
                    label = excluded.label,
                    date = excluded.date,
                    category = excluded.category,
                    frequency = excluded.frequency,
                    medium_price = excluded.medium_price,
                    rating = excluded.rating""",
                    (row['Item'], row['buff_price'], row['market_price'], row['c5_price'], row['market_c5'], row['market_buff'], row['best_price'], row['label'], row['Date'], row['Category'], row['Frequency'], row['Medium Price'], row['Rating'], row['Item'] + ' ' + str(row['Date']))
                )
    conn.commit()
    print('direct has been loaded')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()

try:
    for filename in os.listdir(os.path.dirname(os.path.abspath(__file__))):
        if filename == f'reverse_{timestamp}.csv':
            df = pd.read_csv(filename, encoding='utf-16')
            df['Date'] = pd.to_datetime(df['Date'], format='%d-%m-%Y').dt.strftime('%Y-%m-%d')
            for index, row in df.iterrows():
                cursor.execute(
                    """INSERT INTO public.reverse (item, market_price, buff_price, coef, date, id) 
                    VALUES (%s, %s, %s, %s, %s, %s)
                    on conflict(id)
                    do update set
                    item = excluded.item,
                    market_price = excluded.market_price,
                    buff_price = excluded.buff_price,
                    coef = excluded.coef,
                    date = excluded.date""",
                    (row['Item'], row['market_price'], row['buyorders_price'], row['coef'], row['Date'], row['Item'] + ' ' + str(row['Date']))
                )
    conn.commit()
    print('reverse has been loaded')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()

cursor.close()
conn.close()
```
</details>

### Очистка устаревших данных
После каждого цикла проверяем данные, удаляем записи старше 60 дней

<details>
  <summary><strong>📜 Полный код скрипта</strong></summary>

```python
import psycopg2

conn = psycopg2.connect( 
    host=
    database=
    user=
    password=
)
    
cursor = conn.cursor()

try:
    cursor.execute("""
    delete from market where date < now() - interval'61 DAYS';
    delete from c5 where date < now() - interval'61 DAYS';
    delete from buff_buyorders where date < now() - interval'61 DAYS';
    delete from buff_sell where date < now() - interval'61 DAYS';
    delete from direct where date < now() - interval'61 DAYS';
    delete from reverse where date < now() - interval'61 DAYS';             
    """)
    print('данные очищены')
except Exception as e:
    print(f'возникла ошибка {e}')
    conn.rollback()
finally:
    cursor.close()
    conn.close()
```
</details>  
<br></br>  

## 4.AWS  
### Создание VPC
При создании обязательно включаем расшифровку DNS, т к без этого не получится подключится через внутренний endpoint S3, чего поначалу я не знал. Формат endpoint - com.ap-southeast... и т д, т е для его маршрутизации нужен DNS
<details>
  <summary><strong>🖼️ VPC</strong></summary>

  ![VPC](https://raw.githubusercontent.com/sazhiromru/images/main/VPC%200.PNG)
  ![VPC](https://raw.githubusercontent.com/sazhiromru/images/main/VPC%20settings.PNG)
  ![VPC](https://raw.githubusercontent.com/sazhiromru/images/main/VPC%20creation.PNG)
  ![VPC](https://raw.githubusercontent.com/sazhiromru/images/main/VPC%20scheme.PNG)
  ![VPC](https://raw.githubusercontent.com/sazhiromru/images/main/DNS%20resolution.PNG)
</details>  

### Подключение S3  
Для загрузки файлов я использовал scp через Amazon Cli, но для выгрузки и просмотра результата работы скриптов использовал S3. Для этого создаем точку доступа и маршрут по которому трафик от EC2 до S3 идет через внутренний endpoint

<details>
  <summary><strong>🖼️ S3</strong></summary>

  ![S3](https://raw.githubusercontent.com/sazhiromru/images/main/s3%20endpoint%20creation.PNG)
  ![S3](https://raw.githubusercontent.com/sazhiromru/images/main/s3%20endpoint.PNG)
  ![S3](https://raw.githubusercontent.com/sazhiromru/images/main/s3%20-%20endpoint%20internal%20route.PNG)
  ![S3](https://raw.githubusercontent.com/sazhiromru/images/main/ec2%20s3%20lists.PNG)
  ![S3](https://raw.githubusercontent.com/sazhiromru/images/main/s3%20final.PNG)
</details>    

### Создание RDS и EC2 
1. Стандартные настройки free tier  подходят почти везде, в RDS отключаем auto-scaling, в EC2 - расширернный мониторинг, связываем RDS c EC2 при создании для удобства
2. Настраиваем security groups для неограниченного доступа к EC2 с любого ip
3. Настраиваем security groups для доступа к SQL базе через EC2 

<details>
  <summary><strong>🖼️ RDS и EC2 </strong></summary>

  ![RDS и EC2 ](https://raw.githubusercontent.com/sazhiromru/images/main/RDS%20autoscaling%20off.PNG)
  ![RDS и EC2 ](https://raw.githubusercontent.com/sazhiromru/images/main/aws%20rds%20connect%20to%20ec2.PNG)
  ![RDS и EC2 ](https://raw.githubusercontent.com/sazhiromru/images/main/ec2%20security.PNG)
</details>    

### IAM 
1. Создаем IAM role для EC2 для доступа и управления S3, чтобы можно было использовать консольные команды.
2. Создаем роль для пользователя для настройки AWS CLI

<details>
  <summary><strong>🖼️ IAM </strong></summary>

  ![EC2 role ](https://raw.githubusercontent.com/sazhiromru/images/main/IAM%20EC2%20role%20for%20rds.PNG)
  ![user role](https://raw.githubusercontent.com/sazhiromru/images/main/IAM%20user%20access%20creation.PNG)
</details>    

### Настройка AWS CLI, соединение с EC2 через консоль с PEM ключом
1. Устанавливаем Amazon CLI на windows.
2. Создаем код доступа к Amazon CLI для ранее созданного в IAM пользователя.
3. Получаем CSV файл для созданной ранее роли с данными для входа.
4. Настраиваем доступ пользователя к AMAZON CLI. С помощью pem ключа пытаемся получить доступ через SSH через командную строку. Получаем ошибку, что доступ к pem ключу не ограничен.
5. Убираем разрешения для других пользователей и отключаем наследование разрешений в Windows для этого файла.
6. Логинимся

<details>
  <summary><strong>🖼️ AWS CLI setting </strong></summary>

  ![AWS CLI](https://raw.githubusercontent.com/sazhiromru/images/main/AWS%20CLI%20access%20key.PNG)
  ![AWS CLI](https://raw.githubusercontent.com/sazhiromru/images/main/aws%20cli%20key%20csv.PNG)
  ![AWS CLI](https://raw.githubusercontent.com/sazhiromru/images/main/aws%20cli%20configure.PNG)
  ![AWS CLI](https://raw.githubusercontent.com/sazhiromru/images/main/aws%20pem%20cli%20key%20heritage.PNG)
  ![AWS CLI](https://raw.githubusercontent.com/sazhiromru/images/main/pem%20heritage%20disabled.PNG)
  ![AWS CLI](https://raw.githubusercontent.com/sazhiromru/images/main/aws%20cli%20pem%20solved.PNG)
</details>  

### CLoudwatch 
1. Для отладки смотрим нагрузку CPU при тестировании скриптов. В среднем, если нагрузка держится на 100% больше 10 минут, EC2 перестает функционировать. Поэтому оптимизируем скрипты с учетом показателей CPU.
2. Создаем алерт для EC2 при потери соединения. Ставим автоматическую перезагрузку сервера при потере соединения на 15 минут
3. Создаем алерт для SQL базы данных при загрузке хранилища
4. Подписываемся через SNS на все уведомления

<details>
  <summary><strong>🖼️ Cloudwatch </strong></summary>

  ![Cloudwatch](https://raw.githubusercontent.com/sazhiromru/images/main/cloudwatch%20CPU%20utilization.PNG)
  ![Cloudwatch](https://raw.githubusercontent.com/sazhiromru/images/main/AWS%20cloud%20system%20failure%20alert.PNG)
  ![Cloudwatch](https://raw.githubusercontent.com/sazhiromru/images/main/cloudwatch%20alarms.PNG)
  ![Cloudwatch](https://raw.githubusercontent.com/sazhiromru/images/main/email%20alert%20cloudwatch.PNG)
  ![Cloudwatch](https://raw.githubusercontent.com/sazhiromru/images/main/subscription%20cloudwatch%20confirmed.PNG)
  
</details>   

## Docker
Нам необходимо запустить google chrome на EC2. 
1. Для подбора компонентов и первоначального тестирования создаем контейнер, тестируем на PC, выгружаем на hub.docker, затем запускаем его на EC2.
2. Для структуры файла и ускорения процесса - расширение VSCode для Docker
3.  Библиотеки копируем с аналогичных проектов github

<details>
  <summary><strong>🖼️ Docker</strong></summary>

  ![Docker](https://raw.githubusercontent.com/sazhiromru/images/main/docker%20load.PNG)
  ![Docker](https://raw.githubusercontent.com/sazhiromru/images/main/docker%20pull.PNG)
  ![Docker](https://raw.githubusercontent.com/sazhiromru/images/main/Docker%20install.PNG)
  ![Docker](https://raw.githubusercontent.com/sazhiromru/images/main/docker%20run.PNG)
</details>

<details>
  <summary><strong>📜 Полный код скрипта</strong></summary>

```python
# For more information, please refer to https://aka.ms/vscode-docker-python
FROM python:3-slim

#библиотеки
RUN apt-get update && apt-get install -y \
    wget \
    unzip \
    curl \ 
    fonts-liberation \
    libnss3 \
    libxss1 \
    libasound2 \
    libatk-bridge2.0-0 \
    libgtk-3-0 \
    libdrm2 \
    libgbm1 \
    python3-distutils 

#хром
RUN wget -q https://dl.google.com/linux/direct/google-chrome-stable_current_amd64.deb && \
    dpkg -i google-chrome-stable_current_amd64.deb || apt-get -f install -y && \
    rm google-chrome-stable_current_amd64.deb

RUN CHROME_DRIVER_VERSION=$(curl -sS chromedriver.storage.googleapis.com/LATEST_RELEASE) && \
    wget -q "https://chromedriver.storage.googleapis.com/${CHROME_DRIVER_VERSION}/chromedriver_linux64.zip" && \
    unzip chromedriver_linux64.zip -d /usr/local/bin/ && \
    rm chromedriver_linux64.zip

ENV DISPLAY=:99

COPY requirements.txt .
RUN python -m pip install -r requirements.txt

WORKDIR /app
COPY . /app


RUN adduser -u 5678 --disabled-password --gecos "" appuser && chown -R appuser /app
USER appuser

CMD ["python3", "/app/c5game/c5game/spiders/c5game.py"]
```
</details>
