# Projeto_AplicadoSem_01
Repositório do Conteúdo do projeto aplicado do curso de Ciência de Dados

# Preparação 
import numpy as np
import pandas as pd
from zipfile import ZipFile
import re
import unidecode
import requests as req
from IPython.display import clear_output
from unidecode import unidecode
import os

# Tratamento dos dados da Receita Federal

uniao_estabelecimentos = []

for i in range(0,10):
    caminho = [f'nome_da_pasta/Estabelecimentos{i}', f'nome_da_pasta_02/Estabelecimentos{i}']
        
    if os.path.exists(caminho[0]):
        caminho = caminho[0]
    else:
        caminho = caminho[1]

    arquivo_alvo = os.listdir(caminho)
    
    arquivo = pd.read_csv(f'{caminho}/{arquivo_alvo[0]}',  sep=';', header=None, encoding = 'Latin-1', dtype='unicode', low_memory=False)

    clear_output(wait=True)   

    arquivo[0] = arquivo[0].astype('string').str.pad(width=8, side='left', fillchar='0')
    arquivo[1] = arquivo[1].astype('string').str.pad(width=4, side='left', fillchar='0')
    arquivo[2] = arquivo[2].astype('string').str.pad(width=2, side='left', fillchar='0')
    arquivo['CNPJ'] = arquivo[0].map(str) + arquivo[1].map(str) + arquivo[2].map(str)
    arquivo['CNPJ'] = arquivo['CNPJ'].astype('string').str.pad(width=14, side='left', fillchar='0')
    arquivo = arquivo.drop([1,2,3,6,7,8,9,28,29], axis=1)    
    arquivo = arquivo[(arquivo[19] == 'CE') & (arquivo[5] == "02") & (arquivo[20] == "2304400")]#2304400 obtido do site do IBGE*
    #*https://www.ibge.gov.br/cidades-e-estados/ce/fortaleza.html
    uniao_estabelecimentos.append(arquivo)

uniao_estabelecimentos = pd.concat(uniao_estabelecimentos)

#Renomeando as colunas
uniao_estabelecimentos = uniao_estabelecimentos.rename(columns={0: 'CNPJ.BÁSICO', 13:'TIPO.LOGRADOURO', 14:'LOGRADOURO', 15:'NÚMERO', 16:'COMPLEMENTO', 17:'BAIRRO', 18:'CEP', 19:'UF', 20:'MUNICÍPIO'})

uniao_estabelecimentos.drop_duplicates(subset='CNPJ', keep='first', inplace=True)

uniao_estabelecimentos['NOME.MUNICIPIO'] = 'FORTALEZA' #Novo atributo

uniao_estabelecimentos.info()

# Geocodificação

import requests

chave = '####'

uniao_estabelecimentos = pd.read_csv(r'F:\correios\OneDrive - Correios.com.br\GEDEM\09_Power_BI\base_pbi.csv', sep=';')
uniao_estabelecimentos = uniao_estabelecimentos[uniao_estabelecimentos['NOME.MUNICIPIO'] == 'FORTALEZA']
uniao_estabelecimentos = uniao_estabelecimentos[['CNPJ', 'TIPO.LOGRADOURO', 'LOGRADOURO', 'NÚMERO', 'COMPLEMENTO','BAIRRO', 'CEP', 'UF', 'COD.IBGE.MUNICIPIO', 'NOME.MUNICIPIO', 'Longitude', 'Latitude', 'DIS.KM', 'AGENCIA.PRONEG']]
#uniao_estabelecimentos['DIS.KM'] = uniao_estabelecimentos['DIS.KM'].astype('float')
uniao_estabelecimentos.rename({'AGENCIA.PRONEG':'AGENCIA_PROX'}, axis=1, inplace=True)
uniao_estabelecimentos.head(3)

uniao_estabelecimentos.info()

def enderecoApi(tp_logra, logra, num_logra, bairro_logra, munic_logra, cep_logra, uf_logra):
    enderecamento = ''
    if (num_logra != 0 and num_logra != 'SN') and munic_logra != 'ND':
        enderecamento = f'{num_logra} {tp_logra} {logra} {bairro_logra} {munic_logra} {cep_logra} {uf_logra}'
    elif (num_logra == 0 or num_logra == 'SN') and munic_logra != 'ND':
        enderecamento = f'{tp_logra} {logra} {bairro_logra}, {munic_logra} {cep_logra} {uf_logra}'
    elif munic_logra == 'ND':
        enderecamento = f'{tp_logra} {logra} {bairro_logra} {cep_logra} {uf_logra}'
        
    return enderecamento
    
 for i in uniao_estabelecimentos.index: 
    
    if uniao_estabelecimentos['DIS.KM'][i] > 8:
            
        endereco_busca = enderecoApi(uniao_estabelecimentos["TIPO.LOGRADOURO"][i], uniao_estabelecimentos["LOGRADOURO"][i], uniao_estabelecimentos["NÚMERO"][i], uniao_estabelecimentos["BAIRRO"][i], uniao_estabelecimentos["NOME.MUNICIPIO"][i], uniao_estabelecimentos["CEP"][i], uniao_estabelecimentos["UF"][i])
        
        res = requests.get(f'https://geocode.search.hereapi.com/v1/geocode?q={endereco_busca}&lang=pt-BR&apiKey={chave}')
        
        cidade = 'ND'
        lat = None
        lon = None
        precisao_uf = 0
        precisao_cidade = 0
        precisao_cep = 0
        
        if len(res.json()['items']) > 0:
            if len(res.json()['items'][0]) > 0:
                precisao_uf = res.json()['items'][0]['scoring']['fieldScore'].get('state', 0)
                precisao_cidade = res.json()['items'][0]['scoring']['fieldScore'].get('city', 0)
                precisao_cep = res.json()['items'][0]['scoring']['fieldScore'].get('postalCode', 0)
                cidade = res.json()['items'][0]['address'].get('city', 'ND')
                lat = res.json()['items'][0]['position'].get('lat', False)
                lon = res.json()['items'][0]['position'].get('lng', False)                       
                
                if uniao_estabelecimentos['NOME.MUNICIPIO'][i] == 'ND' and (precisao_uf > 0.7 and precisao_cidade > 0.7):
                    uniao_estabelecimentos['NOME.MUNICIPIO'][i] = cidade.upper()
                    uniao_estabelecimentos['Latitude'][i] = lat
                    uniao_estabelecimentos['Longitude'][i] = lon

                elif uniao_estabelecimentos['NOME.MUNICIPIO'][i] == 'ND' and (precisao_uf > 0.7 and precisao_cep > 0.7):
                    uniao_estabelecimentos['NOME.MUNICIPIO'][i] = cidade.upper()
                    uniao_estabelecimentos['Latitude'][i] = lat
                    uniao_estabelecimentos['Longitude'][i] = lon
                    
                else:
                    uniao_estabelecimentos['Latitude'][i] = lat
                    uniao_estabelecimentos['Longitude'][i] = lon
                    
# Medidas de Distâncias

from haversine import haversine
import re

agencias = pd.read_excel(r'H:\OneDrive - Instituto Presbiteriano Mackenzie\02_PA\Agencias_Correios_Fortaleza.xlsx')
agencias.head()

agencias.to_excel(r'H:\OneDrive - Instituto Presbiteriano Mackenzie\02_PA\Coordenadas_Agencias_Correios_Fortaleza.xlsx', index=False)
agencias.head(2)

agencias_sel = [] #lista de agências selecionada respectivamente para cada instituição
dist = [] #lista das distâncias mínimas selecionadas respectivamente para cada instituição

for j in uniao_estabelecimentos['CNPJ'].index:    
    
    if uniao_estabelecimentos['DIS.KM'][j] > 8: # retirar condição

        inst_coords = (float(uniao_estabelecimentos['Latitude'][j]), float(uniao_estabelecimentos['Longitude'][j]))

        distancia = 10**100

        agencia = None

        for i in agencias.index:

            agencia_coords = (float(agencias['LAT'][i]), float(agencias['LON'][i]))
            calculo = haversine(agencia_coords, inst_coords)  # distância haversine em km
            
            if   distancia >  calculo:
                distancia = calculo
                agencia = agencias['Agência'][i]
                
        uniao_estabelecimentos['DIS.KM'][j], uniao_estabelecimentos['AGENCIA_PROX'][j] = distancia, agencia #retirar


uniao_estabelecimentos.head(3)

uniao_estabelecimentos.info()

# Métricas

from scipy.stats import kurtosis
from scipy.stats import skew

a = agencias_fla[['AGENCIA_PROX', 'DIS.KM']].groupby('AGENCIA_PROX').agg(['mean','median', 'std','var', 'count', 'min', 'max']).values
a = pd.DataFrame(a)
a['UNIDADE'] = lista_agencias
a.rename({0:'Média', 1:'Mediana', 2:'Desvio-padrão', 3:'Variância', 4:'QTD INST', 5:'Mínimo', 6:'Máximo'}, axis=1, inplace=True)
a = a[['UNIDADE', 'QTD INST','Média', 'Mediana', 'Desvio-padrão', 'Variância', 'Mínimo','Máximo']]
a['Amplitude'] = a['Máximo'] - a['Mínimo']
a['Coeficiente de Variação'] = round(a['Desvio-padrão']/a['Média'], 2)

assimetrias = []
curtoses = []

for i in lista_agencias:
    lista_unidade = agencias_fla[agencias_fla['AGENCIA_PROX'] == i]['DIS.KM']
    curtose = kurtosis(lista_unidade)
    curtoses.append(curtose)
    assimetria = skew(lista_unidade)
    assimetrias.append(assimetria)
    
a['Assimetria/Skew'] = assimetrias
a['Curtose (Fisher)'] = curtoses

a.sort_values(by='QTD INST', ascending=False, ignore_index=True, inplace=True)

a = a.append({'UNIDADE' : 'GERAL' , 'QTD INST' : a['QTD INST'].sum(), 'Média' : agencias_fla['DIS.KM'].mean(), 'Mediana': agencias_fla['DIS.KM'].median() , 'Desvio-padrão':agencias_fla['DIS.KM'].std(), 'Variância': agencias_fla['DIS.KM'].var(), 'Mínimo': agencias_fla['DIS.KM'].min(), 'Máximo': agencias_fla['DIS.KM'].max(), 'Amplitude': agencias_fla['DIS.KM'].max() - agencias_fla['DIS.KM'].min(), 'Coeficiente de Variação': round(agencias_fla['DIS.KM'].std()/agencias_fla['DIS.KM'].mean(), 2), 'Assimetria/Skew': skew(agencias_fla['DIS.KM']), 'Curtose (Fisher)':kurtosis(agencias_fla['DIS.KM'])} , ignore_index=True)

a

# Gráficos

import matplotlib.pyplot as plt
import seaborn as sns
import statistics
%matplotlib inline
