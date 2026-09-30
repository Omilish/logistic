# 📚 Metadata Датасета: FeCom Inc. E-com Marketplace


<!-- #region -->
## 1. Общие сведения об источниках
. **FeCom Inc. E-com Marketplace Orders Data CRM** (Kaggle) – 8 файлов, успешно загружены и проанализированы.


## 2. Схема данных: FeCom Inc. E-com Marketplace
*Типы данных указаны как "Наблюдаемые" на основе превью. Для подтверждения на полном объеме датасета необходимо выполнить указанные Code Checks.*

### 2.1. `Orders.csv` (Гранулярность: Заказ)
| Поле | Наблюдаемый тип | Бизнес-семантика (строго из названия/превью) | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Order_ID` | String (UUID-like) | Уникальный идентификатор заказа | `assert df['Order_ID'].is_unique` |
| `Customer_Trx_ID` | String (UUID-like) | Идентификатор клиента/транзакции | `assert df['Customer_Trx_ID'].isin(customers_df['Customer_Trx_ID']).all()` |
| `Order_Status` | String | Текущий статус заказа | `assert set(df['Order_Status'].unique()).issubset({'delivered', 'canceled', 'processing', 'unavailable'})` |
| `Order_Purchase_Timestamp`| String (`YYYY-MM-DD HH:MM`) | Дата и время создания заказа | `assert pd.to_datetime(df['Order_Purchase_Timestamp'], errors='coerce').notna().all()` |
| `Order_Approved_At` | String (`YYYY-MM-DD HH:MM`) | Дата и время подтверждения заказа | Проверка логики: `(df['Order_Approved_At'] >= df['Order_Purchase_Timestamp']).all()` (с учетом NaT) |
| `Order_Delivered_Carrier_Date`| String (`YYYY-MM-DD HH:MM`) | Дата передачи заказа перевозчику | Допускается `NaT` для статусов `canceled`/`unavailable`. Проверить: `df.loc[df['Order_Status']=='delivered', 'Order_Delivered_Carrier_Date'].isna().sum() == 0` |
| `Order_Delivered_Customer_Date`| String (`YYYY-MM-DD HH:MM`) | Дата фактической доставки клиенту | Проверка логики: `(df['Order_Delivered_Customer_Date'] >= df['Order_Delivered_Carrier_Date']).all()` (для не-NULL) |
| `Order_Estimated_Delivery_Date`| String (`YYYY-MM-DD 00:00`) | Ожидаемая дата доставки (SLA) | Проверка формата: `pd.to_datetime(df['Order_Estimated_Delivery_Date'], errors='coerce').notna().all()` |

### 2.2. `Order Items.csv` (Гранулярность: Позиция в заказе)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Order_ID` | String | Внешний ключ к `Orders.csv` | `assert df['Order_ID'].isin(orders_df['Order_ID']).all()` |
| `Order_Item_ID` | Integer | Порядковый номер позиции в заказе | Проверка на уникальность композиции: `assert df.groupby(['Order_ID', 'Order_Item_ID']).size().max() == 1` |
| `Product_ID` | String (UUID-like) | Идентификатор товара | `assert df['Product_ID'].isin(products_df['Product_ID']).all()` |
| `Seller_ID` | String (UUID-like) | Идентификатор продавца | `assert df['Seller_ID'].isin(sellers_df['Seller_ID']).all()` |
| `Shipping_Limit_Date` | String (`YYYY-MM-DD HH:MM`) | Крайний срок отгрузки продавцом | Проверка логики: `>= Order_Purchase_Timestamp` |
| `Price` | Float | Стоимость товара | `assert (df['Price'] >= 0).all()` |
| `Freight_Value` | Float | Стоимость доставки данной позиции | `assert (df['Freight_Value'] >= 0).all()` |

### 2.3. `Products.csv` (Гранулярность: Товар)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Product_ID` | String (UUID-like) | Уникальный идентификатор товара | `assert df['Product_ID'].is_unique` |
| `Product_Category_Name`| String | Название категории товара | **Аномалия:** В превью найдено `#N/A`. Проверка: `df['Product_Category_Name'].isin(['#N/A', 'N/A', 'NaN']).any()`. Действие: заменить на `'Unknown'`. |
| `Product_Weight_Gr` | Integer | Физический вес товара в граммах | `assert (df['Product_Weight_Gr'] >= 0).all()` |
| `Product_Length_Cm` | Integer | Длина упаковки в см | `assert (df['Product_Length_Cm'] >= 0).all()` |
| `Product_Height_Cm` | Integer | Высота упаковки в см | `assert (df['Product_Height_Cm'] >= 0).all()` |
| `Product_Width_Cm` | Integer | Ширина упаковки в см | `assert (df['Product_Width_Cm'] >= 0).all()` |

### 2.4. `Sellers List.csv` (Гранулярность: Продавец)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Seller_ID` | String (UUID-like) | Уникальный идентификатор продавца | `assert df['Seller_ID'].is_unique` |
| `Seller_Name` | String | Название компании-продавца | Проверка на дубликаты имен при разных ID: `df.groupby('Seller_Name')['Seller_ID'].nunique().max()` |
| `Seller_Postal_Code`| String | Почтовый индекс продавца | Проверка формата (напр., `DE-14469`): `df['Seller_Postal_Code'].str.match(r'^[A-Z]{2}-\d+$').all()` |
| `Seller_City` | String | Город продавца | Проверка на пропуски: `df['Seller_City'].isna().sum() == 0` |
| `Country_Code` | String (2 буквы) | Код страны продавца | `assert df['Country_Code'].str.len().eq(2).all()` |
| `Seller_Country` | String | Страна продавца | Проверка соответствия коду страны. |

### 2.5. `Customers.csv` (Гранулярность: Клиент)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Customer_Trx_ID` | String (UUID-like) | Уникальный идентификатор клиента | `assert df['Customer_Trx_ID'].is_unique` |
| `Subscriber_ID` | String (UUID-like) | Внутренний ID подписчика | Проверка на пропуски. |
| `Subscribe_Date` | String (`YYYY-MM-DD`) | Дата регистрации/подписки | `pd.to_datetime(df['Subscribe_Date'], errors='coerce').notna().all()` |
| `First_Order_Date` | String (`YYYY-MM-DD`) | Дата первого заказа | Проверка логики: `>= Subscribe_Date` |
| `Customer_Postal_Code`| String | Почтовый индекс клиента | Проверка формата (напр., `FR-75005`). |
| `Customer_City` | String | Город клиента | Проверка на пропуски. |
| `Customer_Country` | String | Страна клиента | Проверка на пропуски. |
| `Customer_Country_Code`| String (2 буквы) | Код страны клиента | `assert df['Customer_Country_Code'].str.len().eq(2).all()` |
| `Age` | Integer | Возраст клиента | `assert (df['Age'] >= 18).all()` (предположение о дееспособности, требует подтверждения бизнесом). |
| `Gender` | String | Пол клиента | `assert set(df['Gender'].unique()).issubset({'Male', 'Female', 'Other', 'Unknown'})` |

### 2.6. `Geolocations.csv` (Гранулярность: Почтовый индекс)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Geo_Postal_Code` | String | Почтовый индекс | `assert df['Geo_Postal_Code'].is_unique` |
| `Geo_Lat` | String | Широта | **Аномалия:** Используется запятая как десятичный разделитель (напр., `"51,7000"`). Проверка: `df['Geo_Lat'].str.contains(',').any()`. Действие: `pd.to_numeric(df['Geo_Lat'].str.replace(',', '.'), errors='coerce')` |
| `Geo_Lon` | String | Долгота | **Аномалия:** Аналогично `Geo_Lat`. Требуется замена `,` на `.` и конвертация в Float. |
| `Geolocation_City` | String | Город, соответствующий индексу | Проверка на пропуски. |
| `Geo_Country` | String | Страна, соответствующая индексу | Проверка на пропуски. |

### 2.7. `Payments.csv` (Гранулярность: Платеж по заказу)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Order_ID` | String | Внешний ключ к `Orders.csv` | `assert df['Order_ID'].isin(orders_df['Order_ID']).all()` |
| `Payment_Sequential`| Integer | Номер платежа в рамках заказа (для рассрочек) | `assert (df['Payment_Sequential'] >= 1).all()` |
| `Payment_Type` | String | Способ оплаты (напр., `credit_card`) | `print(df['Payment_Type'].unique())` для составления справочника. |
| `Payment_Installments`| Integer | Количество рассрочек | `assert (df['Payment_Installments'] >= 1).all()` |
| `Payment_Value` | Float | Сумма платежа | `assert (df['Payment_Value'] >= 0).all()` |

### 2.8. `Order_Reviews.csv` (Гранулярность: Отзыв)
| Поле | Наблюдаемый тип | Бизнес-семантика | Требуемая программная проверка (Code Check) |
|---|---|---|---|
| `Review_ID` | String (UUID-like) | Уникальный идентификатор отзыва | `assert df['Review_ID'].is_unique` |
| `Order_ID` | String | Внешний ключ к `Orders.csv` | `assert df['Order_ID'].isin(orders_df['Order_ID']).all()` |
| `Review_Score` | Integer | Оценка от 1 до 5 | `assert df['Review_Score'].between(1, 5).all()` |
| `Review_Comment_Title_En`| String / NaN | Заголовок отзыва (англ.) | Допускаются `NaN`. Проверка доли пропусков: `df['Review_Comment_Title_En'].isna().mean()` |
| `Review_Comment_Message_En`| String / NaN | Текст отзыва (англ.) | Допускаются `NaN`. |
| `Review_Creation_Date` | String (`YYYY-MM-DD 00:00`) | Дата создания отзыва | `pd.to_datetime(..., errors='coerce').notna().all()` |
| `Review_Answer_Timestamp`| String (`YYYY-MM-DD HH:MM`) | Дата ответа на отзыв | Проверка логики: `>= Review_Creation_Date` |


<!-- #endregion -->

```python

```
