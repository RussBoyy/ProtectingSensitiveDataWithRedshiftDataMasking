**Protecting senstive data using Redshift Data Masking**

To meet industry and regulatory requirements, it is essential that your data is process with the correct controls in place.  In the upload we will demonstrate how to dynamically obfuscate information upon data retrieval using Redshifts Data Masking feature.  Using dynamic data masking (DDM) in Amazon Redshift, you can protect sensitive data in your data warehouse. You can manipulate how Amazon Redshift shows sensitive data to the user at query time, without transforming it in the database. You control access to data through masking policies that apply custom obfuscation rules to a given user or role. In that way, you can respond to changing privacy requirements without altering underlying data or editing SQL queries.



Create a table

<code>         
CREATE TABLE credit_card_numbers (
  customer_id INT,
  credit_card TEXT
);</code>

Insert values into table to mask

<code>
INSERT INTO credit_card_numbers
VALUES
  (100, ‘1561651618874544’),
  (100, '5487846546132130'),
  (102, '4848454548777788'),
  (102, '6011878746515613'),
  (102, '6011378662059710'),
  (103, '373611968625635')
;
</code>

Run GRANT to grant permission to use the SELECT statement on the table

<code> GRANT SELECT ON credit_card_numbers TO PUBLIC; </code>

Create two separate users.  This will allow us to switch between the two users to confirm only one user can access the data ad the other can't

<code> CREATE USER regular_user WITH PASSWORD '1234Test!';

CREATE USER analytics_user WITH PASSWORD '1234Test!';
</code>

Create the analytics_role role and grant it to analytics_user. The regular_user does not have a role

<code> CREATE ROLE analytics_role;

GRANT ROLE analytics_role TO analytics_user;
</code>

Next, create a masking policy to apply to the analytics role.

<code> CREATE MASKING POLICY mask_credit_card_full
WITH (credit_card VARCHAR(256))
USING ('000000XXXX0000'::TEXT);
</code>

Create a user-defined function that partially obfuscates credit card data.

<code> CREATE FUNCTION REDACT_CREDIT_CARD (credit_card TEXT)
RETURNS TEXT IMMUTABLE
AS$$ import re
regexp = re.compile("^([0-9]{6})[0-9]{5,6}([0-9]{4})") 
match = regexp.search(credit_card)
 if match != None:
   first = match.group(1)
   last = match.group(2)
else:
first = "000000"
last = "0000"
return "{}XXXXX{}".format(first, last)


.$$ LANGUAGE plpythonu;

</code>

Create a masking policy that applies the REDACT_CREDIT_CARD function

<code>
CREATE MASKING POLICY mask_credit_card_partial
WITH (credit_card VARCHAR(256))
USING (REDACT_CREDIT_CARD(credit_card));

</code>

Confirm the masking policies using the associated system views

<code>
SELECT * FROM svv_masking_policy;

SELECT * FROM svv_attached_masking_policy;   
</code>

Attaching a masking policy. Attach the masking policies to the credit card table.

<code>
ATTACH MASKING POLICY mask_credit_card_full
ON credit_cards(credit_card)
TO PUBLIC;
</code>

<code>

Attach mask_credit_card_partial to the analytics role. Users with the analytics role can see partial credit card information

</code>
<code>

ATTACH MASKING POLICY mask_credit_card_partial
ON credit_cards(credit_card)
TO ROLE analytics_role
PRIORITY 10;
</code>

Confirm the masking policies are applied to the table and role in the associated system view

<code>

SELECT * FROM svv_attached_masking_policy
</code>

Confirm the full masking policy is in place for normal users by selecting from the credit card table as regular_user

<code>
SET SESSION AUTHORIZATION regular_user
</code>

<code>
SELECT * FROM credit_cards
</code>

Confirm the partial masking policy is in place for users with the analytics role by 
selecting from the credit card table as analytics_user

<code>
SET SESSION AUTHORIZATION analytics_user

SELECT * FROM credit_cards

</code>