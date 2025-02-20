# Postgres triggers, views and functions
## Dont excecute the errors
## Create person table
CREATE TABLE person (
    login_name varchar(9) not null primary key,
    display_name text
);

## Insert bad data
INSERT INTO person VALUES (NULL, 'Felonious Erroneous');
ERROR:  null value in column "login_name" violates not-null constraint
DETAIL:  Failing row contains (null, Felonious Erroneous).

INSERT INTO person VALUES ('atoolongusername', 'Felonious Erroneous');
ERROR:  value too long for type character varying(9)

## Alter person table, bad data
ALTER TABLE person 
    ADD CONSTRAINT PERSON_LOGIN_NAME_NON_NULL 
    CHECK (LENGTH(login_name) > 0);

ALTER TABLE person 
    ADD CONSTRAINT person_login_name_no_space 
    CHECK (POSITION(' ' IN login_name) = 0);

INSERT INTO person VALUES ('', 'Felonious Erroneous');
ERROR:  new row for relation "person" violates check constraint "person_login_name_non_null"
DETAIL:  Failing row contains (, Felonious Erroneous).

INSERT INTO person VALUES ('space man', 'Major Tom');
ERROR:  new row for relation "person" violates check constraint "person_login_name_no_space"
DETAIL:  Failing row contains (space man, Major Tom).

## Drop constraints
ALTER TABLE PERSON DROP CONSTRAINT person_login_name_no_space;
ALTER TABLE PERSON DROP CONSTRAINT person_login_name_non_null;

## person_bit() Function
CREATE OR REPLACE FUNCTION person_bit() 
    RETURNS TRIGGER
    SET SCHEMA 'public'
    LANGUAGE plpgsql
    SET search_path = public
    AS '
    BEGIN
    END;
    ';

## person_bit Trigger
CREATE TRIGGER person_bit 
    BEFORE INSERT ON person
    FOR EACH ROW EXECUTE PROCEDURE person_bit();

# EXAMPLE 0
## Implement earlier checks
CREATE OR REPLACE FUNCTION person_bit()
    RETURNS TRIGGER
    SET SCHEMA 'public'
    LANGUAGE plpgsql
    AS $$
    BEGIN
    IF LENGTH(NEW.login_name) = 0 THEN
        RAISE EXCEPTION 'Login name must not be empty.';
    END IF;

    IF POSITION(' ' IN NEW.login_name) > 0 THEN
        RAISE EXCEPTION 'Login name must not include white space.';
    END IF;
    RETURN NEW;
    END;
    $$;

## Bad inserts, friendly message
INSERT INTO person VALUES ('', 'Felonious Erroneous');
ERROR:  Login name must not be empty.

INSERT INTO person VALUES ('space man', 'Major Tom');
ERROR:  Login name must not include white space.

# EXAMPLE 1- Audit Logging
## Create table person_audit
CREATE TABLE person_audit (
    login_name varchar(9) not null,
    display_name text,
    operation varchar,
    effective_at timestamp not null default now(),
    userid name not null default session_user
);

## Update person_bit
CREATE OR REPLACE FUNCTION person_bit()
    RETURNS TRIGGER
    SET SCHEMA 'public'
    LANGUAGE plpgsql
    AS $$
    BEGIN
    IF LENGTH(NEW.login_name) = 0 THEN
        RAISE EXCEPTION 'Login name must not be empty.';
    END IF;

    IF POSITION(' ' IN NEW.login_name) > 0 THEN
        RAISE EXCEPTION 'Login name must not include white space.';
    END IF;

    -- New code to record audits

    INSERT INTO person_audit (login_name, display_name, operation) 
        VALUES (NEW.login_name, NEW.display_name, TG_OP);

    RETURN NEW;
    END;
    $$;


DROP TRIGGER person_bit ON person;

CREATE TRIGGER person_biut 
    BEFORE INSERT OR UPDATE ON person
    FOR EACH ROW EXECUTE PROCEDURE person_bit();

## Hanlde delete
CREATE OR REPLACE FUNCTION person_bdt()
    RETURNS TRIGGER
    SET SCHEMA 'public'
    LANGUAGE plpgsql
    AS $$
    BEGIN

    -- Record deletion in audit table

    INSERT INTO person_audit (login_name, display_name, operation) 
      VALUES (OLD.login_name, OLD.display_name, TG_OP);

    RETURN OLD;
    END;
    $$;
        
CREATE TRIGGER person_bdt 
    BEFORE DELETE ON person
    FOR EACH ROW EXECUTE PROCEDURE person_bdt();

## Inserts and testing
INSERT INTO person VALUES ('dfunny', 'Doug Funny');
INSERT INTO person VALUES ('pmayo', 'Patti Mayonnaise');

SELECT * FROM person;
 login_name |   display_name   
------------+------------------
 dfunny     | Doug Funny
 pmayo      | Patti Mayonnaise
(2 rows)

SELECT * FROM person_audit;
 login_name |   display_name   | operation |        effective_at        |  userid  
------------+------------------+-----------+----------------------------+----------
 dfunny     | Doug Funny       | INSERT    | 2018-05-26 18:48:07.6903   | postgres
 pmayo      | Patti Mayonnaise | INSERT    | 2018-05-26 18:48:07.698623 | postgres
(2 rows)

## Update testing
UPDATE person SET display_name = 'Doug Yancey Funny' WHERE login_name = 'dfunny';

SELECT * FROM person;
 login_name |   display_name    
------------+-------------------
 pmayo      | Patti Mayonnaise
 dfunny     | Doug Yancey Funny
(2 rows)

SELECT * FROM person_audit ORDER BY effective_at;
 login_name |   display_name    | operation |        effective_at        |  userid  
------------+-------------------+-----------+----------------------------+----------
 dfunny     | Doug Funny        | INSERT    | 2018-05-26 18:48:07.6903   | postgres
 pmayo      | Patti Mayonnaise  | INSERT    | 2018-05-26 18:48:07.698623 | postgres
 dfunny     | Doug Yancey Funny | UPDATE    | 2018-05-26 18:48:07.707284 | postgres
(3 rows)

## Delete testing
DELETE FROM person WHERE login_name = 'pmayo';

SELECT * FROM person;
 login_name |   display_name    
------------+-------------------
 dfunny     | Doug Yancey Funny
(1 row)

SELECT * FROM person_audit ORDER BY effective_at;
 login_name |   display_name    | operation |        effective_at        |  userid  
------------+-------------------+-----------+----------------------------+----------
 dfunny     | Doug Funny        | INSERT    | 2018-05-27 08:13:22.747226 | postgres
 pmayo      | Patti Mayonnaise  | INSERT    | 2018-05-27 08:13:22.74839  | postgres
 dfunny     | Doug Yancey Funny | UPDATE    | 2018-05-27 08:13:22.749495 | postgres
 pmayo      | Patti Mayonnaise  | DELETE    | 2018-05-27 08:13:22.753425 | postgres
(4 rows)

# EXAMPLE 2 – Derived Values
## Adding abstract columns
ALTER TABLE person ADD COLUMN abstract TEXT;
ALTER TABLE person ADD COLUMN ts_abstract TSVECTOR;

ALTER TABLE person_audit ADD COLUMN abstract TEXT;

## Modify person_bit function
CREATE OR REPLACE FUNCTION person_bit()
    RETURNS TRIGGER
    LANGUAGE plpgsql
    SET SCHEMA 'public'
    AS $$
    BEGIN
    IF LENGTH(NEW.login_name) = 0 THEN
        RAISE EXCEPTION 'Login name must not be empty.';
    END IF;

    IF POSITION(' ' IN NEW.login_name) > 0 THEN
        RAISE EXCEPTION 'Login name must not include white space.';
    END IF;

    -- Modified audit code to include text abstract

    INSERT INTO person_audit (login_name, display_name, operation, abstract) 
        VALUES (NEW.login_name, NEW.display_name, TG_OP, NEW.abstract);

    -- New code to reduce text to text-search vector

    SELECT to_tsvector(NEW.abstract) INTO NEW.ts_abstract;

    RETURN NEW;
    END;
    $$;

## Test abstract
UPDATE person SET abstract = 'Doug is depicted as an introverted, quiet, insecure and gullible 11 (later 12) year old boy who wants to fit in with the crowd.' WHERE login_name = 'dfunny';

SELECT login_name, ts_abstract  FROM person;

# EXAMPLE 3 – Triggers & Views
## Create view to hide abstract vector
CREATE VIEW abridged_person AS SELECT login_name, display_name, abstract FROM person;

## Insert new abstract
INSERT INTO abridged_person VALUES ('skeeter', 'Mosquito Valentine', 'Skeeter is Doug''s best friend. He is famous in both series for the honking sounds he frequently makes.');


SELECT login_name, ts_abstract FROM person WHERE login_name = 'skeeter';

SELECT login_name, display_name, operation, userid FROM person_audit ORDER BY effective_at;

# EXAMPLE 4 – Summary Values
## Create transaction table
CREATE TABLE transaction (
    login_name character varying(9) NOT NULL,
    post_date date,
    description character varying,
    debit money,
    credit money,
    FOREIGN KEY (login_name) REFERENCES person (login_name)
);

## Add cloumn to person, create function and trigger for transaction
ALTER TABLE person ADD COLUMN balance MONEY DEFAULT 0;

CREATE FUNCTION transaction_bit() RETURNS trigger
    LANGUAGE plpgsql
    SET SCHEMA 'public'
    AS $$
    DECLARE
    newbalance money;
    BEGIN

    -- Update person account balance

    UPDATE person 
        SET balance = 
            balance + 
            COALESCE(NEW.debit, 0::money) - 
            COALESCE(NEW.credit, 0::money) 
        WHERE login_name = NEW.login_name
                RETURNING balance INTO newbalance;

    -- Data validation

    IF COALESCE(NEW.debit, 0::money) < 0::money THEN
        RAISE EXCEPTION 'Debit value must be non-negative';
    END IF;

    IF COALESCE(NEW.credit, 0::money) < 0::money THEN
        RAISE EXCEPTION 'Credit value must be non-negative';
    END IF;

    IF newbalance < 0::money THEN
        RAISE EXCEPTION 'Insufficient funds: %', NEW;
    END IF;

    RETURN NEW;
    END;
    $$;



CREATE TRIGGER transaction_bit 
      BEFORE INSERT ON transaction 
      FOR EACH ROW EXECUTE PROCEDURE transaction_bit();

## testing - dont use $
SELECT login_name, balance FROM person WHERE login_name = 'dfunny';

INSERT INTO transaction (login_name, post_date, description, credit, debit) VALUES ('dfunny', '2018-01-11', 'ACH CREDIT FROM: FINANCE AND ACCO ALLOTMENT : Direct Deposit', NULL, '$2,000.00');

SELECT login_name, balance FROM person WHERE login_name = 'dfunny';

INSERT INTO transaction (login_name, post_date, description, credit, debit) VALUES ('dfunny', '2018-01-17', 'FOR:BGE PAYMENT ACH Withdrawal', '$2780.52', NULL);
ERROR:  Insufficient funds: (dfunny,2018-01-17,"FOR:BGE PAYMENT ACH Withdrawal",,"$2,780.52")

SELECT login_name, balance FROM person WHERE login_name = 'dfunny';

INSERT INTO transaction (login_name, post_date, description, credit, debit) VALUES ('dfunny', '2018-01-17', 'FOR:BGE PAYMENT ACH Withdrawal', '$278.52', NULL);

SELECT login_name, balance FROM person WHERE login_name = 'dfunny';

INSERT INTO transaction (login_name, post_date, description, credit, debit) VALUES ('dfunny', '2018-01-23', 'FOR: ANNE ARUNDEL ONLINE PMT ACH Withdrawal', '$35.29', NULL);

SELECT login_name, balance FROM person WHERE login_name = 'dfunny';

# EXAMPLE 5 - Triggers and Views Redux
## balance can be manually altered
BEGIN;
UPDATE person SET balance = '1000000000.00';

SELECT login_name, balance FROM person WHERE login_name = 'dfunny';

ROLLBACK;

## fix- expose balance column in the view
CREATE OR REPLACE VIEW abridged_person AS
  SELECT login_name, display_name, abstract, balance FROM person;

## create function to rollback direct changes to balance, and trigger
CREATE FUNCTION abridged_person_iut() RETURNS TRIGGER
    LANGUAGE plpgsql
    SET search_path TO public
    AS $$
    BEGIN

    -- Disallow non-transactional changes to balance

      NEW.balance = OLD.balance;
    RETURN NEW;
    END;
    $$;

CREATE TRIGGER abridged_person_iut
    INSTEAD OF UPDATE ON abridged_person
    FOR EACH ROW EXECUTE PROCEDURE abridged_person_iut();

## There will be no effect on the balance now
UPDATE abridged_person SET balance = '1000000000.00';

SELECT login_name, balance FROM abridged_person WHERE login_name = 'dfunny';

# EXAMPLE 6 - Elevated Privileges
## Create new user
CREATE USER eve;

## add basic premmisions
GRANT SELECT,INSERT, UPDATE ON abridged_person TO eve;
GRANT SELECT,INSERT ON transaction TO eve;

## test ass eve
SET SESSION AUTHORIZATION eve;

SELECT * FROM person;
ERROR:  permission denied for relation person

SELECT * from person_audit;
ERROR:  permission denied for relation person_audit

SELECT * FROM abridged_person;

SELECT * FROM transaction;

## transacion insert fails
SET SESSION AUTHORIZATION eve;

INSERT INTO transaction (login_name, post_date, description, credit, debit) VALUES ('dfunny', '2018-01-23', 'ACH CREDIT FROM: FINANCE AND ACCO ALLOTMENT : Direct Deposit', NULL, '$2,000.00');
ERROR:  permission denied for relation person
CONTEXT:  SQL statement "UPDATE person 
        SET balance = 
            balance + 
            COALESCE(NEW.debit, 0::money) - 
            COALESCE(NEW.credit, 0::money) 
        WHERE login_name = NEW.login_name"
PL/pgSQL function transaction_bit() line 6 at SQL statement

## Alter auth to be as superuser
RESET SESSION AUTHORIZATION;
ALTER FUNCTION transaction_bit() SECURITY DEFINER;

SET SESSION AUTHORIZATION eve;

INSERT INTO transaction (login_name, post_date, description, credit, debit) VALUES ('dfunny', '2018-01-23', 'ACH CREDIT FROM: FINANCE AND ACCO ALLOTMENT : Direct Deposit', NULL, '$2,000.00');

SELECT * FROM transaction;

SELECT login_name, balance FROM abridged_person WHERE login_name = 'dfunny';
