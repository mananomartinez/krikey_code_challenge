# Krikey Code Challenge - Jose Martinez

## Part 1: SQL Challenge
 
You’ve been asked to write some SQL queries to report on data in a typical relational database (Postgres). Write statements to answer each of the following questions. See the ​Tables​ section below for ​CREATE​ and sample​ INSERT ​statements describing the schema of each table.

 The game items can be:
  - food tokens, called spendables, that can be both received and spent. 
 - birds, here called receivables, that can be received by spending those spendables. 
 - We also want to display those game items at pindrops, for which we need a new table `game_item_locations`

### Questions

1. What should be the CREATE statement for a new table game_item_locations that consists of different game_items assigned to locations per user_id?

```SQL
CREATE TABLE game_item_locations(
   user_id uuid,
   game_item_id uuid,
   location_id uuid,
   CONSTRAINT fk_game_item_id
      FOREIGN KEY(game_item_id)
	  REFERENCES game_items(id),
   CONSTRAINT fk_location_id
      FOREIGN KEY(location_id)
	  REFERENCES locations(id)
);
```

2. Insert dummy data for for transactions, locations, game_item_locations (5k rows for each table for 100 different user_ids)


### SOLUTION 
```SQL
CREATE OR REPLACE FUNCTION createDummyData()
    RETURNS void AS $$
BEGIN
    -- empty the tables to avoid unrelated data
    DELETE FROM game_item_locations;
    DELETE FROM locations;
    DELETE FROM transactions;

    -- create and populate temp users table
    CREATE TEMP TABLE users (id uuid PRIMARY KEY);
    
    FOR i IN 1..100 LOOP
        INSERT INTO users (id)
        SELECT MD5(random()::TEXT || ':' || CURRENT_TIMESTAMP)::UUID;
    END LOOP;

    -- populate locations
    FOR i IN 1..5000 LOOP
        INSERT INTO locations (id, geom)
        SELECT
            MD5(random()::TEXT || ':' || CURRENT_TIMESTAMP)::UUID,
            ST_MakePoint(random()::float, random()::float);
    END LOOP;

    -- populate transactions
   FOR i IN 1..5000 LOOP
      INSERT INTO transactions
         (id, user_id, spent, received)
      SELECT transaction_id.md5 AS id, id_for_user.id AS user_id, spendable.json_spendable AS spent, receivable.json_receivable AS received  FROM
        (SELECT MD5(random()::TEXT || ':' || CURRENT_TIMESTAMP)::UUID) AS transaction_id,
        (SELECT id FROM users ORDER BY random() LIMIT 1) AS id_for_user,
        (SELECT json_object_agg(game_item.id, floor(random() * 10 + 1)::int) AS json_spendable FROM (SELECT id FROM game_items WHERE item_type = 'spendable' ORDER BY random() LIMIT 1 ) AS game_item) AS spendable,
        (SELECT json_object_agg(game_item.id, floor(random() * 10 + 1)::int) AS json_receivable FROM (SELECT id FROM game_items WHERE item_type = 'receivable' ORDER  BY random() LIMIT 1) AS game_item) AS receivable;
   END LOOP;

    -- populate game_item_locations
   FOR i IN 1..5000 LOOP
    INSERT INTO game_item_locations (location_id,game_item_id,user_id)
        SELECT location.id AS location_id, game_item.id AS game_item_id, id_for_user.id AS user_id FROM
        (SELECT id FROM locations ORDER BY random() LIMIT 1) AS location,
        (SELECT id FROM game_items ORDER BY random() LIMIT 1) AS game_item,
        (SELECT id FROM users ORDER BY random() LIMIT 1) AS id_for_user;
   END LOOP;

    -- Delete temporary users table
    DELETE FROM users;
    DROP TABLE users;
END;
$$ LANGUAGE plpgsql;
```

3. Which user has received the `​downy woodpecker​` the most often​?

### SOLUTION 
```SQL
    SELECT user_id, SUM(downy.received::INTEGER) AS downy_sum FROM
     (SELECT user_id, t.received ->>gi.id::text AS received 
        FROM game_items AS gi,transactions AS t 
        WHERE t.received ->>gi.id::text IS NOT NULL 
        AND gi.display_name ='downy woodpecker')
     AS downy GROUP BY user_id ORDER BY downy_sum DESC LIMIT 1;
```

4. What are the aggregated counts per item for each user or itemCounts?
 By this itemCount we mean, aggregating the values of transactions.received for each item key and then subtracting the aggregated values of transactions.spent for each item key.
 We call this collection of itemCounts per user the user’s player-state.
 
 The player-state for user id `ba637a13-4ae0-45c0-b36a-06e762b4d46f` for example, looks like:
 ```JSON
    {
        "5c0e1be2-ed5d-4be8-b45f-26961e27b169": ​1​,
        "e6623c34-64b5-47c6-8e7a-cd944f4141ee": ​0​,
        "ac0eff93-59e4-4f7a-b4d4-82fedc7914ca": ​0​,
        "c90636c6-6e72-4da8-af2e-5761eee72a32": ​0​,
        "be13a699-8328-4d82-827c-bdfa9488804a": ​0​, 
        "bfca8ebe-2c47-4fe6-8248-c5f356650380": ​0 
    }
 ```
### SOLUTION 
I spent a lot of time on this one and I could not get it to function. I chose to include it as part of my solution to demonstrate my thought process. 

```SQL
CREATE OR REPLACE FUNCTION calculatePlayerState(t_user_id uuid)
    RETURNS refcursor AS $$
    declare
        game_item_ids uuid[];
        game_item_id uuid;
    BEGIN
        DROP TABLE IF EXISTS temp_player_state;
        CREATE TEMP TABLE temp_player_state (state_user_id uuid, item_id uuid, item_count integer);

        -- find the distinct game item ids
        SELECT array_agg(id::text) INTO game_item_ids FROM game_items;

        --Build up a table with the items per user and their total SUM
        FOREACH game_item_id in ARRAY game_item_ids loop
           INSERT INTO temp_player_state (state_user_id, item_id, item_count)
            SELECT item_result.user_id AS received_user_id,
                   (item_result.item_id).key::uuid AS item_id,
                   SUM(item_result.received_item_count::integer) - SUM(item_result.spent_item_count::integer) AS item_count FROM
                    (SELECT user_id,
                            json_each_text(received) AS item_id,
                            (received->>game_item_id::text) AS received_item_count,
                            (spent->>game_item_id::text) AS spent_item_count  FROM transactions
                            where
                            user_id=t_user_id and
                            received->>game_item_id::text is not null and
                            spent->>game_item_id::text is not null

                    ) AS item_result
            GROUP BY user_id, item_id ORDER BY received_item_count LIMIT 1;
        END LOOP;
        --Create and return player's state
       RETURN QUERY SELECT array_to_string( array_agg(j_results.j_object),',' ) FROM
            (SELECT json_build_object(item_id::text, item_count ) AS j_object FROM
                temp_player_state
            WHERE
                state_user_id=t_user_id
            GROUP BY item_id, item_count
            ) AS j_results;
        --Create a JSON
        DROP TABLE temp_player_state;
    END;
$$ LANGUAGE plpgsql;
```

5. What are the item_locations within a 3km radius for a given lat, lng for a user_id, sorted by distance?

```SQL
CREATE OR REPLACE FUNCTION findItemsInRadius(longitude float8, latitude float8, given_user_id uuid)
RETURNS SETOF UUID AS $$
    DECLARE
        radius_in_km integer := 3;
        radius_km_from_miles float4  := radius_in_km  * 1609.34;
BEGIN
    RETURN QUERY  SELECT l.id
    FROM game_item_locations as gil
     INNER JOIN  locations as l
     ON l.id = gil.location_id
    WHERE ST_DistanceSphere(geom, ST_MakePoint(longitude,latitude)) <= radius_km_from_miles and
            gil.user_id = given_user_id ORDER BY ST_Distance(geom, ST_MakePoint(longitude,latitude));
END;
$$ LANGUAGE plpgsql;
```

[continue to Part 2](API/README.md)