# Accidents influence on car insurance premium 
 _How do accidents influence car insurance prices?_

### Goal
The goal of this project is to better understand the correlation between car accidents and car premiums. With Python and Tableau, this study will allow for a deeper understanding of how car insurance pricing is influenced by the number of car accidents.  

### Software Used
[Python](https://www.python.org/downloads/)\
[Jupyter Notebook](https://jupyter.org/install)\
[MySql](https://www.mysql.com/downloads/)\
[Tableau](https://www.tableau.com/support/releases)

### Setup

```import pandas as pd
import numpy as np

#Web scraping
import requests
from bs4 import BeautifulSoup as bs
from selenium import webdriver
from selenium.webdriver.common.by import By
from selenium.webdriver.common.action_chains import ActionChains
from selenium.webdriver.support.ui import WebDriverWait
from selenium.webdriver.support import expected_conditions as EC

#Socrata Open Data API
from sodapy import Socrata

#SQL and database handling
from sqlalchemy import create_engine
import pymysql
import mysql.connector

#Certificates
import certifi

#Jupyter extensions
import jupytab

#System handling
import sys

#Time handling
import time
```


#### Grabbing data from _"NYC OPENDATA"_ via API

```
client = Socrata("data.cityofnewyork.us", None)
results = client.get("h9gi-nx95", limit=2000000)
results_df = pd.DataFrame.from_records(results)
```

#### Cleaning Data - Getting rid of the location column
```
for i in results:
    if 'location' in i:
        del i['location']
        
results = [i for i in results if i.get('zip_code') and i['zip_code'].strip()]

results_df = pd.DataFrame.from_records(results)
```

### Create tables in MySql and input data 

```
#Creating Database and Table 
opendata.execute("CREATE DATABASE API")
table = """CREATE TABLE accidents(crash_date DATE, crash_time TIME, on_street_name VARCHAR(255), off_street_name VARCHAR(255), number_of_persons_injured INT, number_of_persons_killed INT, number_of_pedestrians_injured INT, number_of_pedestrians_killed INT, number_of_cyclist_injured INT, number_of_cyclist_killed INT, number_of_motorist_injured INT, number_of_motorist_killed INT, contributing_factor_vehicle_1 VARCHAR(255), contributing_factor_vehicle_2 VARCHAR(255), collision_id INT, vehicle_type_code1 VARCHAR(255), vehicle_type_code2 VARCHAR(255), borough VARCHAR(255), zip_code INT, latitude FLOAT, longitude FLOAT, location VARCHAR(255), cross_street_name VARCHAR(255), contributing_factor_vehicle_3 VARCHAR(255), vehicle_type_code_3 VARCHAR(255), contributing_factor_vehicle_4 VARCHAR(255), vehicle_type_code_4 VARCHAR(255), contributing_factor_vehicle_5 VARCHAR(255), vehicle_type_code_5 VARCHAR(255))"""
opendata.execute(table)

table = """CREATE TABLE insuranceprice(zipcode INT, price INT)"""
opendata.execute(table)

#input data
engine = create_engine("mysql+pymysql://root:[Input_Password]!@localhost/api")
results_df.to_sql('accidents', con=engine, if_exists='append', index=False)
```

### Filtering data in MySQL via python 

```
#SQL query to find the quantity of each unique zipcode
sql = "SELECT zip_code, COUNT(*) AS count FROM api.accidents WHERE zip_code is not NULL GROUP by zip_code ORDER BY count DESC;"

#SQL query to find distinct zipcode
sql = "SELECT DISTINCT zip_code FROM api.accidents WHERE zip_code is not NULL GROUP by zip_code ORDER BY zip_code DESC"
opendata.execute(sql)
results = opendata.fetchall()
results = [item for (item,) in results]
```

### Scraping car insurance price from [car insurance website](https://www.carinsurance.com/calculators/average-car-insurance-rates.aspx)

```
driver = webdriver.Chrome()
driver.get("https://www.carinsurance.com/state/how-much-is-car-insurance-in-new-york/")

for x in results:
    element = driver.find_element(By.ID, "zip-tool-zip")
    element.clear()
    element.send_keys(x)
    time.sleep(2)
    
    element = driver.find_element(By.XPATH, '//input[@type="submit" and @value="Update"]')
    actions = ActionChains(driver)
    actions.move_to_element(element).click().perform()
    time_out_in_seconds = 10
    loading_image = (By.ID, "loading image ID")
    wait = WebDriverWait(driver, time_out_in_seconds)
    wait.until(EC.invisibility_of_element_located(loading_image))
    element.click()
    
    page_source = driver.page_source
    soup = bs(page_source, 'html.parser')
    price = soup.find("div", {"class": "avg_premium"})
    price = price.text.replace("$", '')
    dec = [x, price]
    sql = "INSERT INTO insuranceprice (zipcode, price) VALUES (%s,%s)"
    opendata.execute(sql,dec)
    cnx.commit()
```

### Visual from Tableau showing the number of accidents per zip code 
![Screenshot 2024-06-11 193315](https://github.com/hali-siew/car_insurance/assets/109008466/148e20ea-e7e8-4bf8-87e3-92d9119ce293)

### Filtered Visual Tableau showing the number of accidents per zip code
![image](https://github.com/hali-siew/car_insurance/assets/109008466/6ca0d9a9-de8b-4f82-8b33-5a848fc9ec89)

### Visual from Tableau showing the price of car insurance premium
![Screenshot 2024-06-11 193326](https://github.com/hali-siew/car_insurance/assets/109008466/af23286e-5eeb-405d-bb41-4d63d8a38a16)

## Conclusion 
The visual indicates a strong correlation between the number of accidents and car insurance premiums, but it also reveals that other factors influence pricing. When we filter out zip codes with nearly double the number of accidents, we observe that areas like lower Brooklyn have higher premiums due to a higher number of accidents. Conversely, in Rockaway Beach, high premiums despite low accident rates suggest additional factors at play. Overall, the data highlights that while accident rates significantly impact premiums, they are not the sole determining factor.
