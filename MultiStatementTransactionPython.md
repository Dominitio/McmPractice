Multi-Statement Transactions and Upsert Examples in Python

*Reference:* https://docs.aws.amazon.com/qldb/latest/developerguide/driver.best-practices.html
*Intention:* To add a multi-code tab selection and expand the Running multiple statements per transaction section.


Multi-Statement Insertions within a Transaction

```Python

document_id = ""

# Multi-line insert transaction
def insert_multiple_documents(transaction_executor):
    # Registering John for a new insurance policy
    print("Registering a new person for auto insurance and making sure the vehicle is insured.")
    doc_people = {'firstName': "John",
                  'lastName': "Doe",
                  'age': 32,
                  }
    cursor = transaction_executor.execute_statement("INSERT INTO People ?", doc_people)
    first_record = next(cursor, None)
    if first_record:
        global document_id
        document_id = first_record["documentId"]
        print(f"Inserted John with document Id: {document_id}")

    # Create a new vehicle record for John
    doc_audi = {'brand': "Audi",
                'model': "A4",
                'vin': "3C4PDCBG7ET280296",
                'insured': "true",
                'personId': document_id
                }
    # Register the vehicle under John
    transaction_executor.execute_statement("INSERT INTO Vehicles ?", doc_audi)

```
    

Upserts within a Transaction

```Python
# Insures John's vehicle(s).
# An example UPSERT workflow where it will modify the insurance of any of
# John's existing vehicles or adds and insures a new vehicle owned by John.
def upsert_vehicles(transaction_executor, doc_vehicle):
    # Query the Vehicles table using the vin
    vin = doc_vehicle["vin"]
    cursor = transaction_executor.execute_statement(
        "SELECT brand, model, vin, insured, personId FROM Vehicles WHERE vin = ?", vin)
    first_record = next(cursor, None)
    if first_record:
        # Update the car's insurance status if existing
        insured = doc_vehicle["insured"]
        transaction_executor.execute_statement("UPDATE Vehicles SET insured = ? WHERE vin = ?", insured, vin)
    else:
        # Insert the new vehicle record if not existing
        transaction_executor.execute_statement("INSERT INTO Vehicles ?", doc_vehicle)

...

# John bought a new vehicle Rivian, so he's going
# to unregister the Audi and register the Rivian.

# Update vehicle Audi for not being registered anymore
vehicle_audi = {'brand': "Audi",
                'model': "A4",
                'vin': "3C4PDCBG7ET280296",
                'insured': "false",
                'personId': document_id
                }
qldb_driver.execute_lambda(lambda executor: upsert_vehicles(executor, vehicle_audi))

# Update vehicle Rivian as a new vehicle and registered
vehicle_rivian = {'brand': "Rivian",
                  'model': "R1S",
                  'vin': "4A3AE85H61E295903",
                  'insured': "true",
                  'personId': document_id
                  }
qldb_driver.execute_lambda(lambda executor: upsert_vehicles(executor, vehicle_rivian))
```
