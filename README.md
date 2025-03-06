# scdf-101

### Manual Local Setup

<<<<<<< HEAD
#### Setup RabbitMQ
```
docker run -d  --rm --name rabbitmq -p 5552:5552 -p 15672:15672 -p 5672:5672  \
    -e RABBITMQ_SERVER_ADDITIONAL_ERL_ARGS='-rabbitmq_stream advertised_host localhost' \
    rabbitmq:4-management
```

=======
>>>>>>> ee6746d (SCDF setup on local)
#### SCDF Server 
```
wget https://repo.maven.apache.org/maven2/org/springframework/cloud/spring-cloud-dataflow-server/2.11.5/spring-cloud-dataflow-server-2.11.5.jar

wget https://repo.maven.apache.org/maven2/org/springframework/cloud/spring-cloud-dataflow-shell/2.11.5/spring-cloud-dataflow-shell-2.11.5.jar
```

#### SCDF Skipper
```
wget https://repo.maven.apache.org/maven2/org/springframework/cloud/spring-cloud-skipper-server/2.11.5/spring-cloud-skipper-server-2.11.5.jar
```
#### Run SCDF Skipper
```
java -jar spring-cloud-skipper-server-2.11.5.jar
```
#### Run SCDF Server
```
java -jar spring-cloud-dataflow-server-2.11.5.jar
```
#### Run SCDF Shell
```
java -jar spring-cloud-dataflow-shell-2.11.5.jar
```
<<<<<<< HEAD

#### Access SCDF Server Dashboard
```
http://localhost:9393/dashboard
```
=======
>>>>>>> ee6746d (SCDF setup on local)

sd=rabbit --queues=salesOrder --port=5432  | jdbc --password=password --username=admin --url=jdbc:postgresql://localhost:5432/postgres --table-name=public.salesorders

sd=rabbit --queues=salesOrderQuorum --port=5432 | jdbc --password=password --username=admin --url=jdbc:postgresql://localhost:5432/postgres --table-name=public.salesorders


quorum-jdbc=rabbit --queues=salesOrderQuorum --port=5432 --app.rabbit.spring.cloud.stream.rabbit.bindings.input.consumer.dlqQuorum.enabled=true   --app.rabbit.spring.cloud.stream.rabbit.bindings.input.consumer.quorum.enabled=true | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders"  --app.jdbc.spring.cloud.stream.rabbit.bindings.output.producer.quorum.enabled=true --app.jdbc.spring.cloud.stream.rabbit.bindings.input.producer.dlqQuorum.enabled=true



 --spring.cloud.stream.rabbit.bindings.input.consumer.acknowledgeMode=MANUAL --spring.cloud.stream.rabbit.bindings.input.consumer.dlqQuorum.enabled=true --spring.cloud.stream.rabbit.bindings.input.consumer.autoBindDlq=true --spring.cloud.stream.rabbit.bindings.output.producer.quorum.enabled=true --spring.cloud.stream.rabbit.bindings.input.consumer.quorum.enabled=true  

--app.http-request.spring.cloud.stream.rabbit.bindings.input.consumer.dlqQuorum.enabled=true --app.http-request.spring.cloud.stream.rabbit.bindings.input.consumer.quorum.enabled=true

api=http --port=7575 --path-pattern=timeout --spring.cloud.stream.rabbit.bindings.output.producer.quorum.enabled="true" | http-request  --http.request.body-expression=payload --http.request.http-method-expression="'POST'" --http.request.urlExpression="'http://localhost:8585/timeout'" --expected-response-type=java.lang.String --spring.cloud.stream.rabbit.bindings.input.consumer.acknowledgeMode=MANUAL --spring.cloud.stream.rabbit.bindings.input.consumer.dlqQuorum.enabled=true --spring.cloud.stream.rabbit.bindings.input.consumer.autoBindDlq=true --spring.cloud.stream.rabbit.bindings.output.producer.quorum.enabled=true --spring.cloud.stream.rabbit.bindings.input.consumer.quorum.enabled=true  --http.request.headers-expression="{'Content-Type':'application/json'}" | log --spring.cloud.stream.rabbit.bindings.input.consumer.quorum.enabled="true"

working
```
rabbit --queues=salesOrderQuorumQueue --port=5672 --publisher-confirm-type=CORRELATED  | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```

not working
```
rabbit --queues=salesOrderStream --port=5552 --publisher-confirm-type=CORRELATED  | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" 
```


debug logs
```
rabbit --queues=salesOrderStream --port=5552 --publisher-confirm-type=CORRELATED --logging.level.root=DEBUG | jdbc --password=password --username=admin --url="jdbc:postgresql://localhost:5432/postgres" --table-name="public.salesorders" --logging.level.root=DEBUG
```

	
salesOrderStream



CREATE TABLE salesorders_read (
    order_id SERIAL PRIMARY KEY,
    product VARCHAR(255),
    price DECIMAL(10, 2),
    quantity INTEGER,
    ship_to TEXT,
    payment_method VARCHAR(50),
    order_date TIMESTAMP DEFAULT CURRENT_TIMESTAMP,
);

CREATE TABLE salesorders (
    order_id SERIAL PRIMARY KEY,
    payload VARCHAR(200)
);


## Working example 
CREATE OR REPLACE PROCEDURE copy_sales_orders_regex()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method)
    SELECT
        (regexp_match(payload, E'product=\'([^\']+)\''))[1],
        (regexp_match(payload, E'price=([\\d\\.]+)'))[1]::DECIMAL(10, 2),
        (regexp_match(payload, E'quantity=(\\d+)'))[1]::INTEGER,
        (regexp_match(payload, E'shipTo=\'([^\']+)\''))[1],
        (regexp_match(payload, E'paymentMethod=\'([^\']+)\''))[1]
    FROM salesorders
    WHERE NOT EXISTS (
        SELECT 1
        FROM salesorders_read r
        WHERE r.product = (regexp_match(salesorders.payload, E'product=\'([^\']+)\''))[1]
        AND r.price = (regexp_match(salesorders.payload, E'price=([\\d\\.]+)'))[1]::DECIMAL(10, 2)
        AND r.quantity = (regexp_match(salesorders.payload, E'quantity=(\\d+)'))[1]::INTEGER
        AND r.ship_to = (regexp_match(salesorders.payload, E'shipTo=\'([^\']+)\''))[1]
        AND r.payment_method = (regexp_match(salesorders.payload, E'paymentMethod=\'([^\']+)\''))[1]
    );
END;
$$;

postgres=# CALL copy_sales_orders_regex();
CALL
postgres=# select * from salesorders_read;

CREATE OR REPLACE PROCEDURE insert_sales_orders(input_data TEXT)
LANGUAGE plpgsql
AS $$
DECLARE
    line TEXT;
    order_id INT;
    sales_order_str TEXT;
    product VARCHAR(255);
    price DECIMAL(10, 2);
    quantity INT;
    ship_to TEXT;
    payment_method VARCHAR(50);
BEGIN
    FOR line IN SELECT regexp_split_to_table(input_data, E'\\n') LOOP
        -- Extract order ID and SalesOrder string
        order_id := (regexp_match(line, E'^\\s*(\\d+)\\s*\\|\\s*(.*)$'))[1]::INT;
        sales_order_str := (regexp_match(line, E'^\\s*(\\d+)\\s*\\|\\s*(.*)$'))[2];

        -- Extract product, price, quantity, shipTo, and paymentMethod using regex
        product := (regexp_match(sales_order_str, E'product=\'([^\']+)\''))[1];
        price := (regexp_match(sales_order_str, E'price=([\\d\\.]+)'))[1]::DECIMAL;
        quantity := (regexp_match(sales_order_str, E'quantity=(\\d+)'))[1]::INT;
        ship_to := (regexp_match(sales_order_str, E'shipTo=\'([^\']+)\''))[1];
        payment_method := (regexp_match(sales_order_str, E'paymentMethod=\'([^\']+)\''))[1];

        -- Insert data into the salesorders table
        INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method)
        VALUES (product, price, quantity, ship_to, payment_method);
    END LOOP;
END;
$$;


CREATE OR REPLACE PROCEDURE copy_sales_orders()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method, order_date)
    SELECT product, price, quantity, ship_to, payment_method, order_date
    FROM salesorders
    WHERE NOT EXISTS (
        SELECT 1
        FROM salesorders
        WHERE salesorders_read.product = salesorders.product
        AND salesorders_read.price = salesorders.price
        AND salesorders_read.quantity = salesorders.quantity
        AND salesorders_read.ship_to = salesorders.ship_to
        AND salesorders_read.payment_method = salesorders.payment_method
        AND salesorders_read.order_date = salesorders.order_date
    );
END;
$$;



CREATE OR REPLACE PROCEDURE copy_sales_orders_json()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method)
    SELECT
        (payload->>'product')::VARCHAR(255),
        (payload->>'price')::DECIMAL(10, 2),
        (payload->>'quantity')::INTEGER,
        (payload->>'shipTo')::TEXT,
        (payload->>'paymentMethod')::VARCHAR(50)
    FROM salesorders
    WHERE NOT EXISTS (
        SELECT 1
        FROM salesorders_read r
        WHERE r.product = (salesorders.payload->>'product')::VARCHAR(255)
        AND r.price = (salesorders.payload->>'price')::DECIMAL(10, 2)
        AND r.quantity = (salesorders.payload->>'quantity')::INTEGER
        AND r.ship_to = (salesorders.payload->>'shipTo')::TEXT
        AND r.payment_method = (salesorders.payload->>'paymentMethod')::VARCHAR(50)
    );
END;
$$;



CREATE OR REPLACE PROCEDURE copy_sales_orders_json()
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO salesorders_read (product, price, quantity, ship_to, payment_method)
    SELECT
        ((payload::JSONB)->>'product')::VARCHAR(255),
        ((payload::JSONB)->>'price')::DECIMAL(10, 2),
        ((payload::JSONB)->>'quantity')::INTEGER,
        ((payload::JSONB)->>'shipTo')::TEXT,
        ((payload::JSONB)->>'paymentMethod')::VARCHAR(50)
    FROM salesorders
    WHERE NOT EXISTS (
        SELECT 1
        FROM salesorders_read r
        WHERE r.product = ((salesorders.payload::JSONB)->>'product')::VARCHAR(255)
        AND r.price = ((salesorders.payload::JSONB)->>'price')::DECIMAL(10, 2)
        AND r.quantity = ((salesorders.payload::JSONB)->>'quantity')::INTEGER
        AND r.ship_to = ((salesorders.payload::JSONB)->>'shipTo')::TEXT
        AND r.payment_method = ((salesorders.payload::JSONB)->>'paymentMethod')::VARCHAR(50)
    );
END;
$$;


#