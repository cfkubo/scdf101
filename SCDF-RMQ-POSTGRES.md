### SCDF sample to load data from RabbitMQ queue(quorum/classic) and store the data into postgres table

working to load to postgres table with varchar type
```
rabbit --queues=salesOrderQuorumQueue --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=postgres --username=postgres --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```

```
stream create --name rabbit-to-postgres-json-fixed --definition "rabbit --queues=salesOrderQuorumQueue --port=5672 --publisher-confirm-type=CORRELATED --contentType=application/json | log --name=beforeJdbc | jdbc --password=password --username=admin --url=\"jdbc:postgresql://localhost:5432/postgres\" --table-name=\"public.salesorders_json\" --column-type=payload=json" --deploy
````

```
rabbit --queues=salesOrderQuorumQueue --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=postgres --username=postgres --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders_json" --contentType=application/json --column-type=payload=json
```

not working
```
rabbit --queues=salesOrderStream --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```


debug logs
```
rabbit --queues=salesOrderStream --port=5672 --publisher-confirm-type=CORRELATED --logging.level.root=DEBUG | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" --logging.level.root=DEBUG
```

##### Working psql
```

-- Table: public.salesorders

-- DROP TABLE IF EXISTS public.salesorders;

CREATE TABLE IF NOT EXISTS public.salesorders
(
    order_id integer NOT NULL DEFAULT nextval('salesorders_order_id_seq'::regclass),
    payload character varying(1000) COLLATE pg_catalog."default",
    CONSTRAINT salesorders_pkey PRIMARY KEY (order_id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.salesorders
    OWNER to postgres;

-- Trigger: sales_order_update_trigger

-- DROP TRIGGER IF EXISTS sales_order_update_trigger ON public.salesorders;

CREATE OR REPLACE TRIGGER sales_order_update_trigger
    AFTER INSERT OR UPDATE 
    ON public.salesorders
    FOR EACH ROW
    EXECUTE FUNCTION public.sales_order_trigger_func();

-- Table: public.salesorders_read

-- DROP TABLE IF EXISTS public.salesorders_read;

CREATE TABLE IF NOT EXISTS public.salesorders_read
(
    order_id integer NOT NULL DEFAULT nextval('salesorders_read_order_id_seq'::regclass),
    product character varying(255) COLLATE pg_catalog."default",
    price numeric(10,2),
    quantity integer,
    ship_to character varying(50) COLLATE pg_catalog."default",
    payment_method character varying(50) COLLATE pg_catalog."default",
    order_date date,
    address character varying(50) COLLATE pg_catalog."default",
    store_name character varying(50) COLLATE pg_catalog."default",
    store_address character varying(50) COLLATE pg_catalog."default",
    sales_rep_name character varying(50) COLLATE pg_catalog."default",
    CONSTRAINT salesorders_read_pkey PRIMARY KEY (order_id)
)

TABLESPACE pg_default;

ALTER TABLE IF EXISTS public.salesorders_read
    OWNER to postgres;
	
CREATE OR REPLACE PROCEDURE public.process_sales_order(order_id INT, payload TEXT)
LANGUAGE plpgsql
AS $$
DECLARE
    product VARCHAR(255);
    price NUMERIC(10, 2);
    quantity INT;
    ship_to VARCHAR(50);
    payment_method VARCHAR(50);
    order_date DATE;
    address VARCHAR(50);
    store_name VARCHAR(50);
    store_address VARCHAR(50);
    sales_rep_name VARCHAR(50);
BEGIN
    -- Extract values using regular expressions
    product := REGEXP_REPLACE(payload, '.*product=''([^'']*)''.*', '\1');
    price := REGEXP_REPLACE(payload, '.*price=([\d\.]+).*', '\1')::NUMERIC(10, 2);
    quantity := REGEXP_REPLACE(payload, '.*quantity=(\d+).*', '\1')::INT;
    ship_to := REGEXP_REPLACE(payload, '.*shipTo=''([^'']*)''.*', '\1');
    payment_method := REGEXP_REPLACE(payload, '.*paymentMethod=''([^'']*)''.*', '\1');
    order_date := REGEXP_REPLACE(payload, '.*orderDate=([\d\-]+).*', '\1')::DATE;
    address := REGEXP_REPLACE(payload, '.*address=''([^'']*)''.*', '\1');
    store_name := REGEXP_REPLACE(payload, '.*storeName=''([^'']*)''.*', '\1');
    store_address := REGEXP_REPLACE(payload, '.*storeAddress=''([^'']*)''.*', '\1');
    sales_rep_name := REGEXP_REPLACE(payload, '.*salesRepName=''([^'']*)''.*', '\1');

    -- Insert or update the salesorders_read table
    INSERT INTO public.salesorders_read (
        order_id, product, price, quantity, ship_to, payment_method,
        order_date, address, store_name, store_address, sales_rep_name
    ) VALUES (
        order_id, product, price, quantity, ship_to, payment_method,
        order_date, address, store_name, store_address, sales_rep_name
    )
    ON CONFLICT ((salesorders_read.order_id)) DO UPDATE SET -- Corrected ON CONFLICT syntax
        product = EXCLUDED.product,
        price = EXCLUDED.price,
        quantity = EXCLUDED.quantity,
        ship_to = EXCLUDED.ship_to,
        payment_method = EXCLUDED.payment_method,
        order_date = EXCLUDED.order_date,
        address = EXCLUDED.address,
        store_name = EXCLUDED.store_name,
        store_address = EXCLUDED.store_address,
        sales_rep_name = EXCLUDED.sales_rep_name;

EXCEPTION
    WHEN others THEN
        RAISE NOTICE 'Error processing order %: %', order_id, SQLERRM;
END;
$$;

CREATE OR REPLACE FUNCTION public.sales_order_trigger_func()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Call the stored procedure to process the order data
    CALL public.process_sales_order(NEW.order_id, NEW.payload);
    RETURN NEW;
END;
$$;


CREATE OR REPLACE TRIGGER sales_order_update_trigger
AFTER INSERT OR UPDATE ON public.salesorders
FOR EACH ROW
EXECUTE FUNCTION public.sales_order_trigger_func();


delete from salesorders_read;
call  public.sales_order_trigger_func();

UPDATE public.salesorders
SET payload = '{product=''Gold Watch'', price=800.00, quantity=1, shipTo=''Address 16'', paymentMethod=''paypal'', orderDate=2025-04-01, address=''789 Elm Rd, Somewhere, USA, 54321'', storeName=''Luxury Timepieces'', storeAddress=''101 Maple Ln, Somewhere, USA, 98765'', salesRepName=''Jane Smith''}'
WHERE order_id = 1335;


INSERT INTO public.salesorders (payload)
VALUES (
    '{product=''Silver Necklace'', price=250.50, quantity=5, shipTo=''Address 15'', paymentMethod=''credit'', orderDate=2025-03-10, address=''123 Oak Street, Anytown, USA, 12345'', storeName=''Local Jewelry'', storeAddress=''456 Pine Ave, Anytown, USA, 67890'', salesRepName=''John Doe''}'
);

SELECT * FROM public.salesorders_read
ORDER BY order_id ASC LIMIT 100

DROP PROCEDURE public.process_sales_order(INT, TEXT);
```



```
drop table salesorders;
CREATE TABLE salesorders (
    order_id SERIAL PRIMARY KEY,
    payload VARCHAR(1000)
);

CREATE TABLE salesorders_json (
    order_id SERIAL PRIMARY KEY,
    payload JSON
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

CREATE OR REPLACE PROCEDURE copy_sales_orders_regex()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method, order_date, address, store_name, store_address, sales_rep_name)
    SELECT
        (regexp_match(payload, E'"product":"([^"]+)"'))[1],
        (regexp_match(payload, E'"price":([\\d\\.]+)'))[1]::DECIMAL(10, 2),
        (regexp_match(payload, E'"quantity":(\\d+)'))[1]::INTEGER,
        (regexp_match(payload, E'"shipTo":"([^"]+)"'))[1],
        (regexp_match(payload, E'"paymentMethod":"([^"]+)"'))[1],
        TO_TIMESTAMP((regexp_match(payload, E'"orderDate":(\\d+)'))[1]::BIGINT / 1000.0),
        (regexp_match(payload, E'"address":"([^"]+)"'))[1],
        (regexp_match(payload, E'"storeName":"([^"]+)"'))[1],
        (regexp_match(payload, E'"storeAddress":"([^"]+)"'))[1],
        (regexp_match(payload, E'"salesRepName":"([^"]+)"'))[1]
    FROM salesorders
    WHERE NOT EXISTS (
        SELECT 1
        FROM salesorders_read r
        WHERE r.product = (regexp_match(payload, E'"product":"([^"]+)"'))[1]
        AND r.price = (regexp_match(payload, E'"price":([\\d\\.]+)'))[1]::DECIMAL(10, 2)
        AND r.quantity = (regexp_match(payload, E'"quantity":(\\d+)'))[1]::INTEGER
        AND r.ship_to = (regexp_match(payload, E'"shipTo":"([^"]+)"'))[1]
        AND r.payment_method = (regexp_match(payload, E'"paymentMethod":"([^"]+)"'))[1]
        AND r.order_date = TO_TIMESTAMP((regexp_match(payload, E'"orderDate":(\\d+)'))[1]::BIGINT / 1000.0)
        AND r.address = (regexp_match(payload, E'"address":"([^"]+)"'))[1]
        AND r.store_name = (regexp_match(payload, E'"storeName":"([^"]+)"'))[1]
        AND r.store_address = (regexp_match(payload, E'"storeAddress":"([^"]+)"'))[1]
        AND r.sales_rep_name = (regexp_match(payload, E'"salesRepName":"([^"]+)"'))[1]
    );
END;
$$;


-- Create a trigger to execute the procedure on INSERT or UPDATE
CREATE OR REPLACE FUNCTION trigger_copy_sales_orders_regex()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Execute the procedure
    CALL copy_sales_orders_regex();
    RETURN NEW;
END;
$$;

-- Attach the trigger to the salesorders table
CREATE OR REPLACE TRIGGER salesorders_insert_update_trigger
AFTER INSERT OR UPDATE ON salesorders
FOR EACH STATEMENT
EXECUTE FUNCTION trigger_copy_sales_orders_regex();

```

```
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
```