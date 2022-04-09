# XGBRegressorForecasting
Forecasting the demand using XGB Regressor

# Importing Libraries
import pandas as pd
import numpy as np
import matplotlib.pyplot as plt
import seaborn as sns
%matplotlib inline

from sklearn.metrics import r2_score
from sklearn.metrics import mean_squared_error
from sklearn.metrics import accuracy_score
from sklearn.metrics import mean_absolute_error
from sklearn.metrics import mean_absolute_percentage_error

from xgboost import XGBRegressor as XGB
from sklearn.model_selection import cross_val_score

import warnings
warnings.filterwarnings('ignore')

# Importing Dataset
master_data = pd.read_csv('/content/IDF_train 2.csv', parse_dates=['date'], index_col=['date']).drop(['store'], axis=1)

# Data Wrangling
md = master_data.copy()
md = md.groupby(['item','date']).agg({'sales':'sum'}).reset_index()
md = md.set_index('date', drop=True)

md = md.groupby(['item']).sales.rolling(90).sum().shift(-90).dropna().reset_index().set_index('date', drop=True)
md = md.reset_index()

# Visualizations
for i in list(md['item'].unique()):
  plotdf = md[md['item']==i]
  plotdf = plotdf.drop(['item'], axis=1)
  print('Item_'+str(i))
  print(plotdf,'\n')

  plt.figure(figsize=(6,3))
  plt.plot(plotdf)
  plt.title('Actual Sales - Item {}'.format(i), size=12)
  plt.xlabel('Date', size=10)
  plt.ylabel('Sales #', size=10)
  plt.show()
  
# Feature Engineering
md['day_of_month'] = md['date'].dt.day
md['month'] = md['date'].dt.month
md['year'] = md['date'].dt.year
md['wk_of_year'] = md['date'].dt.weekofyear
md['quarter'] = md['date'].dt.quarter
md['is_month_start'] = md['date'].dt.is_month_start.astype(int)
md['is_month_end'] = md['date'].dt.is_month_end.astype(int)
md['is_quarter_start'] = md['date'].dt.is_quarter_start.astype(int)
md['is_quarter_end'] = md['date'].dt.is_quarter_end.astype(int)

# Correlation matrix for all items
for i in list(md['item'].unique()):
  corr_mat = md[md['item']==i].drop(['item'], axis=1)
  plt.figure(figsize=(8,6))
  plt.title('Item_'+str(i))
  sns.heatmap(corr_mat.corr(), annot=True, fmt='.2f')

# dictionary for collecting the test data forecast results
test_forecast_dict = dict()

# dictionary for collecting the evaluation metrics of the test forecast
test_em_dict = dict()

# running a for loop with a list of unique items in the dataset as the iterable
for i in list(md['item'].unique()):

  # filtering the data item-wise
  split_data = md[md['item']==i].drop(['item'], axis=1)
  #print(split_data)
  
  # Splitting the dataset into dependent and independent variable
  x = split_data.drop(['sales'], axis=1)
  y = split_data['sales']
  #print('Item_'+str(i))
  #print(x,'\n')
  #print(y,'\n')
  
  # Splitting data into training and testing sets
  # Here, we are limiting the test data upto 02/10/2017 because item no 50 is missing 90 days of data as a result of
  # rolling(90), shift(-90) and dropna() Hence, extending the prediction upto 31/12/2017 is resulting in a Value Error.
  
  x_train = x.loc[:'2016-12-31']
  x_test = x.loc['2017-01-01':'2017-10-02'] 
  y_train = y.loc[:'2016-12-31']
  y_test = y.loc['2017-01-01':'2017-10-02']

  #print(len(x_train))
  #print(len(x_test))
  #print(len(y_train))
  #print(len(y_test))

  #print('Item_'+str(i))
  #print(x_train,'\n')
  #print(x_test,'\n')
  #print(y_train,'\n')
  #print(y_test,'\n')

  # Model Building
  # Xtreme Gradient Boosting

  # Initializing the model
  xgbr = XGB(verbosity=0)
  
  # Fitting the data to the model
  xgbr.fit(x_train, y_train)

  # computing training score
  train_score = round(xgbr.score(x_train, y_train),4)
  cvs_train = cross_val_score(xgbr, x_train, y_train,cv=10)
  #print("Training score - {}".format(train_score))
  #print('Mean cv score - ',round(cvs_train.mean(),4),'\n')

  # predicting the test data
  y_pred = xgbr.predict(x_test)

  # creating key and value for test_forecast dictionary
  k = 'Item_'+str(i) #key
  v = list(y_pred) #value
  
  # adding elements to the dictionary
  test_forecast_dict.update({k:v})

  # Statistics
  Mean = round(split_data['sales'].mean(),4)
  Std_dev = round(split_data['sales'].std(),4)

  # Evaluation Metrics
  r2 = round(r2_score(y_test, y_pred),4)
  mse = round(mean_squared_error(y_test, y_pred),4)
  rmse = round(np.sqrt(mse),4)
  mape = round(mean_absolute_percentage_error(y_test, y_pred))

  # creating a key and value for test_forecast evaluation metrics dictionary
  key = 'Item_'+str(i)
  value = {'r2':[r2], 'MSE':[mse], 'RMSE':[rmse], 'MAPE':[mape]}
  test_em_dict.update({key:value})

  #print('Dataset Statistics')
  #print('Mean - {}'.format(Mean))
  #print('Std dev - {}'.format(Std_dev),'\n')
  #print('Evaluation Metrics')
  #print('r2_score - {}'.format(r2))
  #print('MSE - {}'.format(mse))
  #print('RMSE - {}'.format(rmse))
  #print('MAPE - {}'.format(mape),'\n')
  
# Evaluation Metrics
for key, value in test_em_dict.items():
  key = pd.DataFrame(value, index=[key], columns=['r2','MSE','RMSE','MAPE'])
  print(key)
  
# Visualizing the forecast
for k,v in test_forecast_dict.items():
  tfp = pd.DataFrame(v, index=pd.date_range(start='2017-01-01', end='2017-10-02'), columns=[k])
  plt.figure(figsize=(7,4))
  plt.title('Test Forecast - {}'.format(k))
  plt.xlabel('Date', size=10)
  plt.ylabel('Sales #', size=10)

  plt.plot(tfp, label='Forecast Sales')
  plt.plot(split_data.sales, label='Actual Sales')
  plt.xticks(rotation=45)
  plt.legend(loc='best')
  plt.show()
