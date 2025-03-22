
```
insert-to-pg=rabbit --queues=salesOrderQuorumQueue --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=postgres --username=postgres --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```

```
CREATE SEQUENCE IF NOT EXISTS public.salesorders_order_id_seq;

CREATE TABLE IF NOT EXISTS public.salesorders (
    order_id integer NOT NULL DEFAULT nextval('salesorders_order_id_seq'::regclass),
    payload character varying(1000) COLLATE pg_catalog."default",
    CONSTRAINT salesorders_pkey PRIMARY KEY (order_id)
);

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
    AFTER INSERT OR UPDATE 
    ON public.salesorders
    FOR EACH ROW
    EXECUTE FUNCTION public.sales_order_trigger_func();

CREATE TABLE IF NOT EXISTS public.salesorders_read
(
    order_id integer,
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
);

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

CREATE TABLE IF NOT EXISTS public.salesorders_fraud
(
    order_id integer,
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
    CONSTRAINT salesorders_fraud_pkey PRIMARY KEY (order_id)
);


<!-- SELECT order_id, product, price, quantity, ship_to, payment_method, order_date::date AS order_date, address, store_name, store_address, sales_rep_name FROM salesorders_read WHERE payment_method != ''cash'' and (product ilike ''%ring%'' or product = ''diamonds'');

SELECT count(*) FROM public.salesorders;
SELECT count(*) FROM public.salesorders_read;
select count(*) from public.salesorders_fraud; -->

```

```
cdc-fruad-geode=cdc-debezium --cdc.name=postgres-connector --cdc.config.database.dbname=postgres --connector=postgres --cdc.config.database.server.name=my-app-connector --cdc.config.database.user=postgres --cdc.config.database.password=postgres --cdc.config.database.hostname=localhost --cdc.config.database.port=5432 --cdc.flattening.enabled="true" --cdc.config.schema.include.list=public --cdc.config.table.include.list="public.salesorders_read" | geode --host-addresses=localhost:10334 --region-name=orders --key-expression="payload.getField('order_id')" --json="true"
```




