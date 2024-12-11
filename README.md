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

</details>

## 2. Обработка данных
<a id="data-wrangling"></a>
Сбор данных с трех веб-сайтов и сохранение в формате CSV.
  
  Используемые технологии: Pandas, NumPy, Selenium, Beautiful Soup, re
  
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

</details>

### Создание ссылок для проверки рекомендаций.
120 предметов с потенциально лучшей прибылью проверяются по истории продаж за последний месяц. Для этого необходимо сгенерировать ссылку для перехода на страницу предмета на сайта market-csgo.
В соответствии со структурой данных на сайте создан справочник, позволяющий сконструировать ссылку по названию предмета.

Из инетересного - символы '™' и '★'.

Для генерации категорий и корректным сравнением со справочником символы удаляются специальной функцией.

Для генерации ссылки символы меняются на специальные плейсхолдеры и обратно, для результата в виде AK-47/StatTrak™%20AK-47 или Shadow%20Daggers/★%20Shadow%20Daggers
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
</details>
### Проверка 120 предметов по созданным ссылкам
На данном этапе наша задача извлечь со 120 страниц данные по которым строится график продаж предмета, цену запроса на покупку, и уточнить цену продажи.
<details>
  <summary><strong>🖼️ Страница предмета</strong></summary>

  ![Внешний вид сайта](https://raw.githubusercontent.com/sazhiromru/images/main/item%20page.PNG)
</details>
