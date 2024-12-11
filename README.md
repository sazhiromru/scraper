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

