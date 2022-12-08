# NationalAddressByShortAddress
Get the NationalAddress By ShortAddress and insert it in Sql Server.


import pyodbc
import pandas as pd
import requests
from time import sleep


# Sql server connection detail
cnxn_str = ("Driver={SQL Server Native Client 11.0};"
            "Server=**;"
            "Database=**;"
            "UID=**;"
            "PWD=**;")

# National address URL
format = 'json'
page = '1'
encode = 'utf8&'
# shortaddress = 'MDSB2551'
api_key = '*****'

count_inserted = 0
column = ['BuildingNumber', 'Street', 'District', 'City','PostCode','AdditionalNumber']
data = {
    "totalSearchResults": "1",
    "Addresses": [{
        "BuildingNumber": "8080",
        "Street": "عاصم بن ثابت",
        "District": "حي العوالي",
        "City": "الرياض",
        "PostCode": "14926",
        "AdditionalNumber": "3125"
    }],
    "success": True,
    "statusdescription": "SUCCESS"
}
request_status = {'statusdescription': ''}
try:
    print('********************START************************')
    # open conn
    try:
        cnxn = pyodbc.connect(cnxn_str)
        cursor = cnxn.cursor()
        print("Connection  opend .. ")
        print('***********************************************')
    except TypeError:
        print("TypeError : Connection .. ")

    # C:\Users\19548\Desktop\short_address.xlsx
    file_path = input('Please Enter File Path : ')
    short_na_excel = pd.read_excel(r''+file_path, sheet_name='Sheet1')
    short_na_excel = short_na_excel.to_dict('records')
    # print(short_na_excel)
    for row in short_na_excel:
        sleep(5)  # Time in seconds
        url = ('https://apina.address.gov.sa/NationalAddress/NationalAddressByShortAddress/NationalAddressByShortAddress?'
               f'format={format}'
               f'&page={page}'
               f'&encode={encode}'
               f'shortaddress={row["short_address"]}'
               f'&api_key={api_key}')

        try:
            response = requests.get(url)
            data = response.json()
            request_status = response.json()
            print("The 'Requests' is finished .. ")
            # print(data)
        except requests.exceptions.RequestException as e:
            print(f"TypeError : Requests .. {e}")

        try:
            # print(request_status)
            if 'statusdescription' in request_status.keys() and request_status['statusdescription'] == 'SUCCESS':
                check = cursor.execute(
                    f'(SELECT COUNT(*) FROM dbo.NA_LONG_ADDRESS  WHERE Customer_Code = \'{row["customer_code"]}\');')
                check = check.fetchone()[0]
                if check >= 1:
                    long_add_tabel = (
                        f'UPDATE [ABP_BMB].[dbo].[NA_LONG_ADDRESS] SET is_active = \'No\' WHERE Customer_Code = \'{row["customer_code"]}\' ;')
                    cursor.execute(long_add_tabel)
                    cnxn.commit()
                # print(data)
                for col in column:
                    if col not in data["Addresses"][0].keys():
                        data["Addresses"][0][col] = ''
                    
                insert_long_add_tabel = ('INSERT INTO [ABP_BMB].[dbo].[NA_LONG_ADDRESS]('
                                         '[BuildingNumber],'
                                         '[Street],'
                                         '[District],'
                                         '[City],'
                                         '[PostCode],'
                                         '[AdditionalNumber],'
                                         '[Customer_Code],'
                                         '[Short_Address],'
                                         '[Is_Active])'
                                         'VALUES ('
                                         f'\'{data["Addresses"][0]["BuildingNumber"]}\','
                                         f'\'{data["Addresses"][0]["Street"]}\','
                                         f'\'{data["Addresses"][0]["District"]}\','
                                         f'\'{data["Addresses"][0]["City"]}\','
                                         f'\'{data["Addresses"][0]["PostCode"]}\','
                                         f'\'{data["Addresses"][0]["AdditionalNumber"]}\','
                                         f'\'{row["customer_code"]}\','
                                         f'\'{row["short_address"]}\','
                                         f'\'Yes\');')

                cursor.execute(insert_long_add_tabel)
                cnxn.commit()
                count_inserted += 1
                print(f'Insert Succsess .. {row["short_address"]}')
            else:
                print(
                    f'Request Status is {request_status} ... Short Address {row["short_address"]}')
        except TypeError as e:
            print(f"TypeError : Insert or Update Error .. {e}")
    del cnxn  # close the connection
    print('***********************************************')
    print("Connection Closed .. ")
    print(f"Record inserted  = {count_inserted}")
except TypeError:
    print('TypeError Error handling Excel File ..')

