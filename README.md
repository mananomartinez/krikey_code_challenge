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

    insert into users (id)
    select MD5(random()::TEXT || ':' || CURRENT_TIMESTAMP)::UUID
    from generate_series(1, 100) s(i);

    -- populate locations
    FOR i IN 1..5000 LOOP
        insert into locations (id, geom)
        select
            MD5(random()::TEXT || ':' || CURRENT_TIMESTAMP)::UUID,
            ST_MakePoint(random()::float, random()::float);
    END LOOP;

    -- populate transactions
   FOR i IN 1..5000 LOOP
      INSERT INTO transactions
         (id, user_id, spent, received)
      SELECT transaction_id.md5 as id, id_for_user.id as user_id, spendable.json_spendable as spent, receivable.json_receivable as received  FROM
        (SELECT MD5(random()::TEXT || ':' || CURRENT_TIMESTAMP)::UUID) as transaction_id,
        (SELECT id FROM users ORDER BY random() LIMIT 1) AS id_for_user,
        (SELECT json_object_agg(game_item.id, floor(random() * 10 + 1)::int) as json_spendable FROM (SELECT id FROM game_items WHERE item_type = 'spendable' ORDER BY random() LIMIT 1 ) as game_item) AS spendable,
        (SELECT json_object_agg(game_item.id, floor(random() * 10 + 1)::int) as json_receivable FROM (SELECT id FROM game_items WHERE item_type = 'receivable' ORDER  BY random() LIMIT 1) as game_item) as receivable;
   END LOOP;

    -- populate game_item_locations
   FOR i IN 1..5000 LOOP
    INSERT INTO game_item_locations (location_id,game_item_id,user_id)
        SELECT location.id as location_id, game_item.id as game_item_id, id_for_user.id as user_id FROM
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

```SQL
    select user_id, sum(downy.received::INTEGER) as downy_sum from
     (select user_id, t.received ->>gi.id::text as received 
        from game_items as gi,transactions as t 
        where t.received ->>gi.id::text  is not null 
        and gi.display_name ='downy woodpecker')
     as downy group by user_id order by downy_sum desc limit 1;
```

4. What are the aggregated counts per item for each user or itemCounts?
 By this itemCount we mean, aggregating the values of transactions.received for each item key and then subtracting the aggregated values of transactions.spent for each item key.
 We call this collection of itemCounts per user the user’s player-state.

5. What are the item_locations within a 3km radius for a given lat, lng for a user_id, sorted by distance?


[continue to Part 2](API/README.md)