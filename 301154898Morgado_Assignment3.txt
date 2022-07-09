/*
1.	The home page of the Brewbean's Web site has an option for members to log on with their IDs and passwords.
Develop a procedure named MEMBER_CK_SP that accepts the ID and password as inputs, checks whether they make up a valid logon,
and returns the member name and cookie value. The name should be returned as a single text string containing the first and last name.
The head developer wants the number of parameters minimized so that the same parameter is used to accept the password and return the name value.
Also, if the user doesn't enter a valid username and password, return the value INVALID in a parameter named p_check.
Test the procedure using a valid logon first, with the username rat55 and password kile. Then try it with an invalid logon by changing the username to rat.
*/

CREATE OR REPLACE
PROCEDURE MEMBER_CK_SP
    (p_id IN bb_shopper.username%TYPE,
    p_password IN OUT VARCHAR2,
    p_cookie OUT bb_shopper.cookie%TYPE, 
    p_check OUT VARCHAR2)
AS
BEGIN
    SELECT (firstname || ' ' || lastname) name, COOKIE
    INTO p_password, p_cookie
    FROM BB_SHOPPER
    WHERE p_id = username
    AND p_password = password;
    p_check := 'VALID';
    -- DBMS_OUTPUT.PUT_LINE('Name: ' || p_password);
    -- DBMS_OUTPUT.PUT_LINE('Cookie: ' || p_cookie);
EXCEPTION
    WHEN NO_DATA_FOUND THEN
        p_check := '* INVALID *';
        p_password := '';
    -- DBMS_OUTPUT.PUT_LINE('The input information is: ' || p_check);                    
END MEMBER_CK_SP;
/
--------------------------------------------------------------------------

DECLARE
    lv_id VARCHAR2(8) := 'rat55';
    lv_password VARCHAR2(30) := 'kile';
    lv_cookie NUMBER(4);
    lv_check VARCHAR2(11);
BEGIN
    MEMBER_CK_SP(lv_id, lv_password, lv_cookie, lv_check);
    DBMS_OUTPUT.PUT_LINE('Name: ' || lv_password);
    DBMS_OUTPUT.PUT_LINE('Cookie: ' || lv_cookie);
    DBMS_OUTPUT.PUT_LINE('Check: ' || lv_check);
END;
/

/*
2.	Create a procedure named DDPAY_SP that identifies whether a donor currently has a completed pledge with no monthly payments.
A donor ID is the input to the procedure. Using the donor ID, the procedure needs to determine whether the donor has any currently
completed pledges based on the status field and is/was not  on a monthly payment plan. If so, the procedure is to return the Boolean
value TRUE. Otherwise, the value FALSE should be returned. Test the procedure with an anonymous block.
*/
CREATE OR REPLACE
PROCEDURE DDPAY_SP
    (p_id IN DD_DONOR.iddonor%TYPE,
    p_completed OUT BOOLEAN)
AS
    lv_exist NUMBER(4);
BEGIN
    p_completed := FALSE;

    SELECT COUNT(*)
    INTO lv_exist
    FROM dd_donor
    JOIN dd_pledge
    USING(iddonor)
    WHERE iddonor = p_id  
    AND IDSTATUS = 20
    AND PAYMONTHS = 0;
    -- If count not equal 0, p_completed is set to TRUE
    IF lv_exist != 0 THEN
    p_completed := TRUE;
    END IF;
END DDPAY_SP;
/

--------------------------------------------------------------------------

DECLARE
    lv_id NUMBER(4) := 302;
    lv_complete BOOLEAN;
BEGIN
    DDPAY_SP(lv_id, lv_complete);
    IF lv_complete THEN
    DBMS_OUTPUT.PUT_LINE('Pledge completed with NO monthly payments: TRUE');
    ELSE
    DBMS_OUTPUT.PUT_LINE('Pledge completed with NO monthly payments: FALSE');
    END IF; 
END;
/


/*
3.	Create a procedure named DDCKPAY_SP that confirms whether a monthly pledge payment is the correct amount.
The procedure needs to accept two values as input: a payment amount and a pledge ID. Based on these inputs,
the procedure should confirm that the payment is the correct monthly increment amount, based on pledge data in the database.
If it isn't, a custom Oracle error using error number 20050 and the message "Incorrect payment amount - planned payment = ??" should be raised.
The ?? should be replaced by the correct payment amount. The database query in the procedure should be formulated so that no rows are returned
if the pledge isn'ton a monthly payment plan or the pledge isn't found. If the query returns no rows, the procedure should display the message
"No payment information." Test the procedure with the pledge ID 104 and the payment amount $25. Then test with the same pledge ID but the
payment amount $20.Finally,test the procedure with a pledge ID for a pledge that doesn't have monthly payments associated with it.
*/

CREATE OR REPLACE
PROCEDURE DDCKPAY_SP
    (p_pledgeID IN dd_pledge.idpledge%TYPE,
    p_pay_amount IN dd_payment.PAYAMT%TYPE    
    )
AS
    lv_pledge_amt dd_pledge.pledgeamt%TYPE;
    lv_paymonths dd_pledge.paymonths%TYPE;
    lv_pay_amt dd_payment.payamt%TYPE;
BEGIN
    SELECT paymonths, pledgeamt
    INTO lv_paymonths, lv_pledge_amt 
    FROM dd_pledge
    WHERE idpledge = p_pledgeID
    AND paymonths > 0;

    lv_pay_amt := (lv_pledge_amt/lv_paymonths);
    IF p_pay_amount != lv_pay_amt THEN
        RAISE_APPLICATION_ERROR(-20050, 'Incorrect payment ammount - planned payment = ' || lv_pay_amt);
    ELSE
        DBMS_OUTPUT.PUT_LINE('Monthly payment amount of: ' || lv_pay_amt || ' is CORRECT.'); 
    END IF;
EXCEPTION
    WHEN NO_DATA_FOUND
    THEN DBMS_OUTPUT.PUT_LINE('* No payment information *');
END DDCKPAY_SP;
/

--------------------------------------------------------------------------
-- First Test
DECLARE
    lv_pledgeID NUMBER(5) := 104;
    lv_pay_amount NUMBER(8,2) := 25;
BEGIN
    DDCKPAY_SP(lv_pledgeID, lv_pay_amount);
END;
/
-- Second Test
DECLARE
    lv_pledgeID NUMBER(5) := 104;
    lv_pay_amount NUMBER(8,2) := 20;
BEGIN
    DDCKPAY_SP(lv_pledgeID, lv_pay_amount);
END;
/
-- Third Test
DECLARE
    lv_pledgeID NUMBER(5) := 101;
    lv_pay_amount NUMBER(8,2) := 25;
BEGIN
    DDCKPAY_SP(lv_pledgeID, lv_pay_amount);
END;
/


/*
4.	The Reporting and Analysis Department has a database containing tables that hold summarized data
to make report generation simpler and more efficient. They want to schedule some jobs to run nightly
to update summary tables. Create two procedures to update the following tables(assuming that existing
data in the tables is deleted before these procedures run):

Create the PROD_SALES_SUM_SP procedure to update the BB_PROD_SALES
table, which holds total sales dollars and total sales quantity by product ID, month,
and year. The order date should be used for the month and year information.
Create the SHOP_SALES_SUM_SP procedure to update the BB_SHOP_SALES
table, which holds total dollar sales by shopper ID. The total should include only
product amounts—no shipping or tax amounts.
The BB_SHOP_SALES and BB_PROD_SALES tables have already been created. Use the
DESCRIBE command to review their table structures. Run pl/sql block to execute and present results for both procedures.
*/

CREATE OR REPLACE
PROCEDURE PROD_SALES_NUM_SP AS
    prod_sales bb_prod_sales%ROWTYPE;
    CURSOR prod_sales_totals IS
        SELECT idproduct,
            EXTRACT(MONTH FROM dtordered) month,
            EXTRACT(YEAR FROM dtordered) year,
            SUM (bitem.quantity) qty,
            SUM (price * bitem.quantity) total FROM
        bb_basket bas
        JOIN bb_basketitem bitem USING(idbasket)
        WHERE orderplaced = 1
        GROUP BY (idproduct, EXTRACT(MONTH FROM dtordered), EXTRACT(YEAR FROM dtordered))
        ORDER BY EXTRACT(MONTH FROM dtordered), EXTRACT(YEAR FROM dtordered), idproduct;
BEGIN
    OPEN prod_sales_totals;
        LOOP
            FETCH prod_sales_totals INTO prod_sales;
            EXIT WHEN prod_sales_totals%NOTFOUND;
            INSERT INTO bb_prod_sales
            VALUES prod_sales;
        END LOOP;
    CLOSE prod_sales_totals;
END;
/
--------------------------------------------------------------------------

BEGIN
    PROD_SALES_NUM_SP;
END;
/

--------------------------------------------------------------------------

CREATE OR REPLACE
PROCEDURE SHOP_SALES_NUM_SP AS
    shopper_sales bb_shop_sales%ROWTYPE;
    CURSOR shopper_sales_totals IS
        SELECT idshopper, SUM(bitem.price * bitem.quantity) total
        FROM bb_basket b
        JOIN bb_basketitem bitem
        USING(idbasket)
        WHERE orderplaced = 1
        GROUP BY idshopper
        ORDER BY idshopper;
BEGIN
    OPEN shopper_sales_totals;
        LOOP
            FETCH shopper_sales_totals INTO shopper_sales;
            EXIT WHEN shopper_sales_totals%NOTFOUND;
            INSERT INTO bb_shop_sales
            VALUES shopper_sales;
        END LOOP;
    CLOSE shopper_sales_totals;
END;
/
--------------------------------------------------------------------------

BEGIN
    SHOP_SALES_NUM_SP;
END;
/