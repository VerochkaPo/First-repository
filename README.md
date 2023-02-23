# First-repository
import pandas as pd

import numpy as np

import time

import requests

VACANCIES_PER_PAGE = 100

API_AREAS_REQUEST = 'https://api.hh.ru/areas'
API_VACANCIES_REQUEST = 'https://api.hh.ru/vacancies'

CURRANCY_USD = "USD"
CURRANCY_EUR = "EUR"

EXCHANGE_RATE_USD = 71
EXCHANGE_RATE_EUR = 86

TAX_RATE = 0.13
COUNTRY_RUSSIA = 'Россия'

COMMENT_REGION_PROCESSING_START = 'Обрабатываю регионы:'
COMMENT_REGION_PROCESSING_FINISH = 'Готово! Найдено вакансий: '
COMMENT_KEYWORD_PROCESSING_START = 'Ищу вакансии по ключевому слову '

date_from = [('2022-10-31')]
date_to = [('2022-10-31')]

def get_russian_regions():
    ''' Возвращает DataFrame регионов РФ'''
    regions = requests.get(API_AREAS_REQUEST)
    
    region_frame = pd.DataFrame(regions.json())
    
    russia_index = region_frame[region_frame['name'] == COUNTRY_RUSSIA].index.tolist()[0]
    russia_regions_frame = pd.DataFrame(region_frame['areas'][russia_index])
    return russia_regions_frame

def get_region_vacancies_count_data(keyword, region, date_from, date_to):
    ''' Возвращает количество вакансий в регионе region, 
    найденных по ключевому слову keyword,
    '''    

    result = requests.get(
        API_VACANCIES_REQUEST, 
        params = {
            'text': keyword, 
            'per_page' : 1, 
            'search_field' : 'description', 
            'area': region,
            # Добавляем новые параметры поиска
            'date_from': date_from,
            'date_to': date_to,
        }
    )
            
    if(result.status_code == 200):
        return int(result.json()['found'])

    return 0

def get_region_vacancies(keyword, region, page, date_from, date_to):
    response = requests.get(
        API_VACANCIES_REQUEST,
        params = {
            'text': keyword, 
            'area': region, 
            'per_page' : VACANCIES_PER_PAGE, 
            'search_field' : 'description', 
            'page': page,
            # Добавляем новые параметры поиска
            'date_from': date_from,
            'date_to': date_to,
            }
    )
    if(response.status_code == 200):
        found_vacancies = response.json()['items']

        return found_vacancies
    
    return None

def get_vacancy(id):
    
    ''' Возвращает вакансию в виде словаря по ее id'''
    response = requests.get(API_VACANCIES_REQUEST + "/" + str(id))
    
    if(response.status_code == 200):
        return response.json()
    
    return None

def enrich_vacancies_list(vacancies, region_name):
    ''' Обогащает вакансии из vacancies данными из детальной информации по каждой 
    вакансии, а также названием региона region_name'''
    for vacancy in vacancies:
      details = get_vacancy(vacancy['id'])
      if not details['experience']:
        vacancy['experience'] = ""
      else:
        vacancy['experience'] = details['experience']
      if  not details['key_skills']:
        vacancy['key_skills'] = ""
      else:
        vacancy['key_skills'] = details['key_skills'] 
       
        
    
    
      vacancy['region'] = region_name
    return vacancies

def get_vacancies_data_frame(keyword_list, region = None):
    ''' Возвращает DataFrame вакансий, найденных по ключевым словам из списка keyword_list'''
    vacancies_list = []
    
    russia_regions_frame = get_russian_regions()

    if (region is not None):
        russia_regions_list = [region]
    else: 
        russia_regions_list = [x for x in russia_regions_frame['id']]
    
    #print(date_from, date_to)
    for keyword in [keyword_list]:
        
        print(COMMENT_KEYWORD_PROCESSING_START + keyword)
        print(COMMENT_REGION_PROCESSING_START)
            
        for region in russia_regions_list:
            found = get_region_vacancies_count_data(keyword, region, date_from, date_to)
            print(found)
            region_name = russia_regions_frame.loc[russia_regions_frame['id'] == region]['name'].values[0];
            print(region_name)
            for page in range((found // VACANCIES_PER_PAGE) + 1):
                found_vacancies = get_region_vacancies(keyword, region, page, date_from, date_to)
                
                
                found_vacancies = enrich_vacancies_list(found_vacancies, region_name)
                time.sleep(0.25)


                if (found_vacancies is not None):
                    vacancies_list = vacancies_list + found_vacancies

    print(COMMENT_REGION_PROCESSING_FINISH)
    print(len(vacancies_list))
    
    full_vacancies_frame = pd.DataFrame(vacancies_list)
    
    return full_vacancies_frame

df = get_vacancies_data_frame('Разработчик')
df.to_excel("Разработчик.xlsx")
