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

## Visual from Tableau showing the number of accidents per zip code 
![image](https://github.com/hali-siew/car_insurance/assets/109008466/48b0802c-afcc-4bcc-864d-14ec21637a62)

## Visual from Tableau showing the price of car insurance premium
![Screenshot 2024-06-11 193326](https://github.com/hali-siew/car_insurance/assets/109008466/af23286e-5eeb-405d-bb41-4d63d8a38a16)

### Analysis 
Some general observations include the Brooklyn area having a higher number of accidents, which results in higher insurance premiums compared to areas like Manhattan. Additionally, regions with higher premiums often influence surrounding areas, even if those areas do not have similarly high accident rates.

## Filtered Visual showing the number of accidents in Manhattan
![Map 1](https://github.com/hali-siew/car_insurance/assets/109008466/dd2d8cb1-f0d1-44e4-b2e6-4bce04efd9f4)

## Filtered Visual showing the price of car insurance premiums in Manhattan
![Map 1 2 0](https://github.com/hali-siew/car_insurance/assets/109008466/c409019a-87c8-4212-9766-7abdf3e3faf6)

### Analysis 
A closer comparison of the number of accidents and insurance premiums in Manhattan shows a strong correlation. Areas with a high number of accidents tend to have higher premiums. However, some areas with lower accident rates still have high premiums, indicating that other factors also influence the cost of insurance.

## Conclusion 
The visual indicates a strong correlation between the number of accidents and car insurance premiums but also reveals that other factors influence pricing. The overall map shows that areas like Brooklyn have higher accident rates accompanied by higher premiums. Conversely, a closer look at Manhattan shows that while the general trend holds, certain areas do not align with this assumption. Overall, the data highlights that while accident rates significantly impact premiums, they are not the sole determining factor.
