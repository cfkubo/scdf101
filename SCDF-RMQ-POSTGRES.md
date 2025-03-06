### SCDF sample to load data from RabbitMQ queue(quorum/classic) and store the data into postgres table

working
```
rabbit --queues=salesOrderQuorumQueue --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```

not working
```
rabbit --queues=salesOrderStream --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```


debug logs
```
rabbit --queues=salesOrderStream --port=5672 --publisher-confirm-type=CORRELATED --logging.level.root=DEBUG | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" --logging.level.root=DEBUG
```

```
CREATE TABLE salesorders (
    order_id SERIAL PRIMARY KEY,
    payload VARCHAR(200)
);

CREATE TABLE salesorders_read (
    order_id SERIAL PRIMARY KEY,
    product VARCHAR(255),
    price DECIMAL(10, 2),
    quantity INTEGER,
    ship_to VARCHAR(50),
    payment_method VARCHAR(50),
    order_date DATE,
    address VARCHAR(50),
    store_name VARCHAR(50),
    store_address VARCHAR(50),
    sales_rep_name VARCHAR(50)
);

CREATE OR REPLACE PROCEDURE copy_sales_orders_json()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method, order_date, address, store_name, store_address, sales_rep_name)
    SELECT
        (regexp_match(payload, E'product=\'([^\']+)\''))[1],
        (regexp_match(payload, E'price=([\\d\\.]+)'))[1]::DECIMAL(10, 2),
        (regexp_match(payload, E'quantity=(\\d+)'))[1]::INTEGER,
        (regexp_match(payload, E'shipTo=\'([^\']+)\''))[1],
        (regexp_match(payload, E'paymentMethod=\'([^\']+)\''))[1],
        (regexp_match(payload, E'orderDate=([\\d\\-]+)'))[1]::DATE,
        (regexp_match(payload, E'address=\'([^\']+)\''))[1],
        (regexp_match(payload, E'storeName=\'([^\']+)\''))[1],
        (regexp_match(payload, E'storeAddress=\'([^\']+)\''))[1],
        (regexp_match(payload, E'salesRepName=\'([^\']+)\''))[1]
    FROM salesorders
    WHERE NOT EXISTS (
        SELECT 1
        FROM salesorders_read r
        WHERE r.product = (regexp_match(salesorders.payload, E'product=\'([^\']+)\''))[1]
        AND r.price = (regexp_match(salesorders.payload, E'price=([\\d\\.]+)'))[1]::DECIMAL(10, 2)
        AND r.quantity = (regexp_match(salesorders.payload, E'quantity=(\\d+)'))[1]::INTEGER
        AND r.ship_to = (regexp_match(salesorders.payload, E'shipTo=\'([^\']+)\''))[1]
        AND r.payment_method = (regexp_match(payload, E'paymentMethod=\'([^\']+)\''))[1]
        AND r.order_date = (regexp_match(payload, E'orderDate=([\\d\\-]+)'))[1]::DATE
        AND r.address = (regexp_match(payload, E'address=\'([^\']+)\''))[1]
        AND r.store_name = (regexp_match(payload, E'storeName=\'([^\']+)\''))[1]
        AND r.store_address = (regexp_match(payload, E'storeAddress=\'([^\']+)\''))[1]
        AND r.sales_rep_name = (regexp_match(payload, E'salesRepName=\'([^\']+)\''))[1]
    );
END;
$$;

-- Create a trigger to execute the procedure on INSERT or UPDATE
CREATE OR REPLACE FUNCTION trigger_copy_sales_orders_json()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Execute the procedure
    CALL copy_sales_orders_json();
    RETURN NEW;
END;
$$;

-- Attach the trigger to the salesorders table
CREATE OR REPLACE TRIGGER salesorders_insert_update_trigger
AFTER INSERT OR UPDATE ON salesorders
FOR EACH STATEMENT
EXECUTE FUNCTION trigger_copy_sales_orders_json();

```
