# E-Commerce-Database-
E-commerce Database with Full schema, Inserted data, roles and security and all automation triggers





``SQL 
  ``CREATE DATABASE E_commerceDB;
GO

USE E_commerceDB;
GO
---Customer Table--- (Dimension Table)
DROP TABLE IF EXISTS Customers 
CREATE TABLE Customers(
CustomerID INT IDENTITY (1,1) PRIMARY KEY,
FirstName VARCHAR (100) NOT NULL,
LastName VARCHAR(100) NOT NULL,
Email VARCHAR(150) NOT NULL UNIQUE,---I need ya email and don't reuse it ---
Phone VARCHAR(20) UNIQUE,
City VARCHAR(100),
Country VARCHAR DEFAULT 'Nigeria',
CreatedAt DATETIME DEFAULT GETDATE()
);
GO
 ALTER TABLE Customers
 ALTER COLUMN Country VARCHAR(100);
--Product Table-- ( Dimension Table)--

DROP TABLE IF EXISTS Products
CREATE TABLE Products (
PRODUCTID INT IDENTITY (1,1) PRIMARY KEY,
ProductName VARCHAR(100) NOT NULL,
Category VARCHAR (100) NOT NULL,
Price DECIMAL (10,2) NOT NULL CHECK (Price >0),
StockQty INT NOT NULL CHECK (StockQty >0),
CreatedAt DATETIME DEFAULT GETDATE()
);
GO

--Orders Table ( The Fact Table ) 

DROP TABLE IF EXISTS Orders 
CREATE TABLE Orders(
OrderID INT IDENTITY (1,1) PRIMARY KEY,
CustomerID INT NOT NULL,
OrderDate DATETIME DEFAULT GETDATE(),
OrderStatus VARCHAR(50) DEFAULT 'Pending',CHECK (OrderStatus IN ('Pending','Paid','Shipped','Cancelled')),
FOREIGN KEY (CustomerID) REFERENCES Customers(CustomerID)
);
GO

---OrderItems Table( Between Orders & Products)
DROP TABLE IF EXISTS OrderItems
CREATE TABLE OrderItems(
OrderItemID INT IDENTITY (1,1) PRIMARY KEY ,
OrderID INT NOT NULL,
ProductID INT NOT NULL,
Quantity INT NOT NULL CHECK (Quantity >0),
UnitPrice DECIMAL(10,2) NOT NULL CHECK(UnitPrice >0),

FOREIGN KEY (OrderID) REFERENCES Orders(OrderID),
FOREIGN KEY (ProductID) REFERENCES Products(ProductID)
);
GO

---Transactions Table (Payments)
DROP TABLE IF EXISTS Transactions 
CREATE TABLE Transactions (
TransactionID INT IDENTITY(1,1) PRIMARY KEY,
OrderID INT NOT NULL,
Amount DECIMAL(10,2) NOT NULL CHECK (Amount > 0),
PaymentMethod VARCHAR(50) CHECK   (PaymentMethod  IN  ('Card', 'Transfer','Wallet','Cash')),
PaymentStatus VARCHAR(50) DEFAULT 'Pending' CHECK (PaymentStatus IN ('Pending', 'Successful', 'Failed')),
PaidAt DATETIME,
FOREIGN KEY (OrderID) REFERENCES Orders(OrderID)
);

INSERT INTO Customers (FirstName, LastName, Email, Phone, City,Country)
VALUES
('Stanley', 'Bright', 'stanley@gmail.com', '08011112222', ' Lagos State', 'Nigeria'),
('Emeka', 'Bright', 'Emeka@gmail.com', '08033334444', 'Abuja ', 'Nigeria'),
('Prosper', 'Chimaobi', 'Prosper@gmail.com', '08104856237', 'Akwa Ibom', 'Nigeria'),
('Marvelous', 'onwuzirike', 'Marvelous@gmail.com', '09044449999', 'Rivers State', 'Nigeria');

INSERT INTO Products (ProductName, Category, Price, StockQty)
VALUES
('iPhone 15', 'Electronics', 1200000, 10),
('Dell Laptop', 'Computers', 850000, 5),
('Wireless Earbuds', 'Accessories', 30000, 50),
('Charger', 'Accessories', 200000 ,5),
('RingLight', 'Accessories', 450000,10);

INSERT INTO Orders (CustomerID, OrderStatus)
VALUES
(3, 'Pending'),
(5, 'Paid'),
(4, 'Shipped'),
(2, 'Cancelled');



INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
VALUES
(2, 1, 1, 1200000),
(3, 2, 2, 30000),
(4,3,3,50000),
(5,4,5,70000),
(6,5,3,45000);



INSERT INTO Transactions (OrderID, Amount, PaymentMethod, PaymentStatus, PaidAt) 
VALUES
(2, 60000, 'Card', 'Successful', GETDATE()),
(3, 1200000, 'Transfer', 'Pending', GETDATE()),
(4, 45000, 'Wallet', 'Successful', GETDATE()),
(5, 30000, 'Cash', 'Failed', GETDATE()),
(6, 2000, 'Transfer', 'Pending', GETDATE());

UPDATE Transactions ---( I wanted all transaction with pending and failed not to have Timestamp)
SET PaidAt = NULL
WHERE PaymentStatus IN ('Pending', 'Failed');

SELECT *
FROM Transactions;

---Assigning Permission to all the users--
ALTER ROLE db_owner ADD MEMBER ManagerUser; ---I assigned full access to the ManagingDirector---
ALTER ROLE db_datareader ADD MEMBER AnalsytUser;----I assigned Read acees to the analyst---
ALTER ROLE db_datareader ADD MEMBER SalesUser;----I assigned Read access to the Sales Person.

GRANT SELECT ON Customers TO SalesUser;
GRANT SELECT ON Products TO SalesUser;
GRANT SELECT ON Orders TO SalesUser;
GRANT SELECT ON OrderItems TO SalesUser;
GRANT SELECT ON Transactions TO SalesUser;

--Trigger For Automation--
GO
CREATE TRIGGER      PreventTransactionDelete---This Trigger prevents deletion of data from Transaction table.
ON Transactions
INSTEAD OF DELETE 
AS 
PRINT 'You are not permitted to delete data from the transactions table';

CREATE TABLE OrdersUpdate( -----Temporary table for storing the update data on Order table--
OrderID INT,
OrderDate DATE,
ModifyDate DATETIME 
);


GO
CREATE TRIGGER OrdersUpdatedRow ---this trigger will be responsible for filling in a historical table (OrdersUpdate) where information about the updated rows is kept.
ON Orders 
AFTER UPDATE
AS 
INSERT INTO OrdersUpdate(OrderID,OrderDate,ModifyDate)---changes to orders will be saved into OrdersUpdate to be used by the company for auditing purposes.
SELECT OrderID,OrderDate,GETDATE()
FROM inserted;


DROP TRIGGER IF EXISTS OrdersUpdatedRow;

GO
CREATE TRIGGER OrdersUpdatedRow---this trigger will be responsible for filling in a historical table (OrdersUpdate) where information about the updated rows is kept.
ON Orders
AFTER UPDATE
AS 
INSERT INTO OrdersUpdate (OrderID,OrderDate,ModifyDate)---changes to orders will be saved into OrdersUpdate to be used by the company for auditing purposes.
SELECT OrderID,OrderDate,GETDATE()
FROM inserted;

ALTER TABLE OrdersUpdate
ADD ModifiedBy NVARCHAR(100);

DROP TRIGGER IF EXISTS OrderUpdatedRow;
GO 
CREATE TRIGGER OrdersUpdateRow
ON Orders
AFTER UPDATE 
AS
INSERT INTO OrdersUpdate(OrderID, OrderDate,ModifyDate, ModifiedBy)
SELECT OrderID,
OrderDate,
GETDATE(),
SUSER_SNAME() ----Captures the username who made the update 
FROM inserted;
GO 


UPDATE Orders
SET OrderStatus = 'Pending'
WHERE OrderID = 3;

SELECT *
FROM OrdersUpdate;


GO
CREATE OR ALTER TRIGGER OrderItems_AfterInsert_Cleaning
ON OrderItems
AFTER INSERT 
AS 
BEGIN 
SET NOCOUNT ON;
---Prevent endless recursive calls---
IF TRIGGER_NESTLEVEL() >1
RETURN;
EXEC dbo.SP_Cleaning @Table = 'OrderItems';
END 

-- First, create the cleaning stored procedure
GO
CREATE OR ALTER PROCEDURE dbo.SP_Cleaning
    @Table NVARCHAR(128)
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Only process if the table is OrderItems
    IF @Table = 'OrderItems'
    BEGIN
        -- Clean only the recently inserted records using the inserted table
        
        -- 1. Ensure Quantity is valid (must be positive, set to 1 if invalid)
        UPDATE oi
        SET oi.Quantity = CASE 
            WHEN oi.Quantity IS NULL OR oi.Quantity <= 0 THEN 1 
            ELSE oi.Quantity 
        END
        FROM OrderItems oi
        INNER JOIN inserted i ON oi.OrderItemID = i.OrderItemID
        WHERE oi.Quantity IS NULL OR oi.Quantity <= 0;
        
        -- 2. Ensure UnitPrice is valid (must be non-negative, set to 0 if invalid)
        UPDATE oi
        SET oi.UnitPrice = CASE 
            WHEN oi.UnitPrice IS NULL OR oi.UnitPrice < 0 THEN 0 
            ELSE oi.UnitPrice 
        END
        FROM OrderItems oi
        INNER JOIN inserted i ON oi.OrderItemID = i.OrderItemID
        WHERE oi.UnitPrice IS NULL OR oi.UnitPrice < 0;
        
        -- 3. Delete records with invalid foreign keys (optional - only if you don't have FK constraints)
        -- Remove orphaned records where OrderID or ProductID don't exist
        /*
        DELETE oi
        FROM OrderItems oi
        INNER JOIN inserted i ON oi.OrderItemID = i.OrderItemID
        WHERE NOT EXISTS (SELECT 1 FROM Orders o WHERE o.OrderID = oi.OrderID)
           OR NOT EXISTS (SELECT 1 FROM Products p WHERE p.ProductID = oi.ProductID);
        */
        
        -- 4. Remove exact duplicates (same OrderID, ProductID, Quantity, UnitPrice)
        -- Keep only the first occurrence
        WITH CTE_Duplicates AS (
            SELECT oi.OrderItemID,
                   ROW_NUMBER() OVER (
                       PARTITION BY oi.OrderID, oi.ProductID, oi.Quantity, oi.UnitPrice 
                       ORDER BY oi.OrderItemID
                   ) AS RowNum
            FROM OrderItems oi
            INNER JOIN inserted i ON oi.OrderID = i.OrderID
        )
        DELETE FROM CTE_Duplicates
        WHERE RowNum > 1;
        
    END
END
GO

-- Then create the trigger
CREATE OR ALTER TRIGGER OrderItems_AfterInsert_Cleaning
ON OrderItems
AFTER INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    ---Prevent endless recursive calls---
    IF TRIGGER_NESTLEVEL() > 1
        RETURN;
    
    EXEC dbo.SP_Cleaning @Table = 'OrderItems';
END
GO

-- Test with invalid data
INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
VALUES 
    (1, 101, -5, 10.00),      -- Negative quantity (will be set to 1)
    (1, 102, 3, -15.00),      -- Negative price (will be set to 0)
    (1, 103, NULL, 20.00),    -- NULL quantity (will be set to 1)
    (1, 103, 2, 25.00),       -- Valid
    (1, 103, 2, 25.00);       -- Duplicate (will be removed)

 -- Drop the old trigger
DROP TRIGGER IF EXISTS OrderItems_AfterInsert_Cleaning;
GO

-- Create INSTEAD OF trigger
CREATE OR ALTER TRIGGER OrderItems_InsteadOfInsert_Cleaning
ON OrderItems
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert cleaned data directly
    INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
    SELECT 
        OrderID,
        ProductID,
        -- Clean Quantity: ensure it's positive, default to 1
        CASE 
            WHEN Quantity IS NULL OR Quantity <= 0 THEN 1 
            ELSE Quantity 
        END AS Quantity,
        -- Clean UnitPrice: ensure it's non-negative, default to 0
        CASE 
            WHEN UnitPrice IS NULL OR UnitPrice < 0 THEN 0 
            ELSE UnitPrice 
        END AS UnitPrice
    FROM inserted;
    
    -- Remove duplicates if needed (run after insert)
    WITH CTE_Duplicates AS (
        SELECT 
            OrderItemID,
            ROW_NUMBER() OVER (
                PARTITION BY OrderID, ProductID, Quantity, UnitPrice 
                ORDER BY OrderItemID
            ) AS RowNum
        FROM OrderItems oi
        WHERE EXISTS (SELECT 1 FROM inserted i WHERE i.OrderID = oi.OrderID)
    )
    DELETE FROM CTE_Duplicates
    WHERE RowNum > 1;
END
GO

-- Drop the old trigger
DROP TRIGGER IF EXISTS OrderItems_InsteadOfInsert_Cleaning;
GO

-- Create improved INSTEAD OF trigger
CREATE OR ALTER TRIGGER OrderItems_InsteadOfInsert_Cleaning
ON OrderItems
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Insert only valid cleaned data
    INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
    SELECT 
        i.OrderID,
        i.ProductID,
        -- Clean Quantity: ensure it's positive, default to 1
        CASE 
            WHEN i.Quantity IS NULL OR i.Quantity <= 0 THEN 1 
            ELSE i.Quantity 
        END AS Quantity,
        -- Clean UnitPrice: ensure it's non-negative, default to 0
        CASE 
            WHEN i.UnitPrice IS NULL OR i.UnitPrice < 0 THEN 0 
            ELSE i.UnitPrice 
        END AS UnitPrice
    FROM inserted i
    -- Only insert if OrderID exists in Orders table
    WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.OrderID = i.OrderID)
      -- Only insert if ProductID exists in Products table (if you have one)
      -- AND EXISTS (SELECT 1 FROM Products p WHERE p.ProductID = i.ProductID)
    ;
    
    -- Remove duplicates (run after insert)
    WITH CTE_Duplicates AS (
        SELECT 
            OrderItemID,
            ROW_NUMBER() OVER (
                PARTITION BY OrderID, ProductID, Quantity, UnitPrice 
                ORDER BY OrderItemID
            ) AS RowNum
        FROM OrderItems oi
        WHERE EXISTS (SELECT 1 FROM inserted i WHERE i.OrderID = oi.OrderID)
    )
    DELETE FROM CTE_Duplicates
    WHERE RowNum > 1;
END
GO


-- Now test with the data (OrderID 1 exists, so it will work)
INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
VALUES 
    (1, 101, -5, 10.00),      -- Will be cleaned
    (1, 102, 3, -15.00),      -- Will be cleaned
    (1, 103, NULL, 20.00),    -- Will be cleaned
    (1, 103, 2, 25.00),       -- Valid
    (1, 103, 2, 25.00),       -- Duplicate (will be removed)
    (999, 104, 5, 30.00);     -- Invalid OrderID (will be silently skipped)

    -- First, create a log table
CREATE TABLE OrderItems_CleaningLog (
    LogID INT IDENTITY(1,1) PRIMARY KEY,
    LogDate DATETIME DEFAULT GETDATE(),
    Action NVARCHAR(100),
    OrderID INT,
    ProductID INT,
    OriginalQuantity INT,
    CleanedQuantity INT,
    OriginalUnitPrice DECIMAL(10,2),
    CleanedUnitPrice DECIMAL(10,2),
    WasRejected BIT DEFAULT 0
);
GO

-- Update the trigger with logging
CREATE OR ALTER TRIGGER OrderItems_InsteadOfInsert_Cleaning
ON OrderItems
INSTEAD OF INSERT
AS
BEGIN
    SET NOCOUNT ON;
    
    -- Log all incoming data (before cleaning)
    INSERT INTO OrderItems_CleaningLog 
        (Action, OrderID, ProductID, OriginalQuantity, CleanedQuantity, 
         OriginalUnitPrice, CleanedUnitPrice, WasRejected)
    SELECT 
        CASE 
            WHEN NOT EXISTS (SELECT 1 FROM Orders o WHERE o.OrderID = i.OrderID) 
            THEN 'REJECTED - Invalid OrderID'
            WHEN i.Quantity IS NULL OR i.Quantity <= 0 OR i.UnitPrice IS NULL OR i.UnitPrice < 0
            THEN 'CLEANED'
            ELSE 'INSERTED - No cleaning needed'
        END AS Action,
        i.OrderID,
        i.ProductID,
        i.Quantity AS OriginalQuantity,
        CASE 
            WHEN i.Quantity IS NULL OR i.Quantity <= 0 THEN 1 
            ELSE i.Quantity 
        END AS CleanedQuantity,
        i.UnitPrice AS OriginalUnitPrice,
        CASE 
            WHEN i.UnitPrice IS NULL OR i.UnitPrice < 0 THEN 0 
            ELSE i.UnitPrice 
        END AS CleanedUnitPrice,
        CASE 
            WHEN NOT EXISTS (SELECT 1 FROM Orders o WHERE o.OrderID = i.OrderID) 
            THEN 1 
            ELSE 0 
        END AS WasRejected
    FROM inserted i;
    
    -- Insert only valid cleaned data
    INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
    SELECT 
        i.OrderID,
        i.ProductID,
        CASE 
            WHEN i.Quantity IS NULL OR i.Quantity <= 0 THEN 1 
            ELSE i.Quantity 
        END AS Quantity,
        CASE 
            WHEN i.UnitPrice IS NULL OR i.UnitPrice < 0 THEN 0 
            ELSE i.UnitPrice 
        END AS UnitPrice
    FROM inserted i
    WHERE EXISTS (SELECT 1 FROM Orders o WHERE o.OrderID = i.OrderID);
    
    -- Remove duplicates
    WITH CTE_Duplicates AS (
        SELECT 
            OrderItemID,
            ROW_NUMBER() OVER (
                PARTITION BY OrderID, ProductID, Quantity, UnitPrice 
                ORDER BY OrderItemID
            ) AS RowNum
        FROM OrderItems oi
        WHERE EXISTS (SELECT 1 FROM inserted i WHERE i.OrderID = oi.OrderID)
    )
    DELETE FROM CTE_Duplicates
    WHERE RowNum > 1;
    
    -- Log duplicates that were removed
    INSERT INTO OrderItems_CleaningLog (Action, OrderID, ProductID, OriginalQuantity, CleanedQuantity, OriginalUnitPrice, CleanedUnitPrice)
    SELECT 
        'DUPLICATE REMOVED',
        i.OrderID,
        i.ProductID,
        i.Quantity,
        i.Quantity,
        i.UnitPrice,
        i.UnitPrice
    FROM inserted i
    GROUP BY i.OrderID, i.ProductID, i.Quantity, i.UnitPrice
    HAVING COUNT(*) > 1;
END
GO


-- Test insert
INSERT INTO OrderItems (OrderID, ProductID, Quantity, UnitPrice)
VALUES 
    (1, 101, -5, 10.00),
    (1, 102, 3, -15.00),
    (1, 103, NULL, 20.00),
    (1, 103, 2, 25.00),
    (1, 103, 2, 25.00),
    (999, 104, 5, 30.00);  -- Invalid OrderID

-- Check the log to see what was cleaned
SELECT 
    LogDate,
    Action,
    OrderID,
    ProductID,
    OriginalQuantity,
    CleanedQuantity,
    OriginalUnitPrice,
    CleanedUnitPrice,
    WasRejected
FROM OrderItems_CleaningLog
ORDER BY LogID DESC;

SELECT*
FROM  OrderItems;

ALTER TABLE OrderItems 
ADD TotalAmount AS (UnitPrice  * Quantity) ;----I used computed column instead of Trigger to automatically input Total Amount in th OrderItems Table(Sales table)

CREATE TABLE OrderItems_AuditLog (              --- This Audit table tracks every change to OrderItems table (Sales), Records old and new values for price,quantity, and total. Logs who made the change(Username), Logs when the change happened(timestamp), Identifies the type of action(INSERT,UPDATE, DELETE) 
    AuditID INT IDENTITY(1,1) PRIMARY KEY,
    OrderItemID INT,
    OldUnitPrice DECIMAL(10,2) NULL,
    NewUnitPrice DECIMAL(10,2) NULL,
    OldQuantity INT NULL,
    NewQuantity INT NULL,
    OldTotalAmount DECIMAL(18,2) NULL,
    NewTotalAmount DECIMAL(18,2) NULL,
    ActionType VARCHAR(20),       -- INSERT or UPDATE
    ChangedBy VARCHAR(100),       -- Username
    ChangedAt DATETIME DEFAULT GETDATE()  -- When change happened
);

-- Step 1: Drop any existing triggers
DROP TRIGGER IF EXISTS trg_OrderItems_Audit;
DROP TRIGGER IF EXISTS trg_OrderItems_AfterInsert;
GO


CREATE TRIGGER trg_OrderItems_Audit ------This trigger executed what is in the audit table to tracks every change to OrderItems table (Sales), Records old and new values for price,quantity, and total. Logs who made the change(Username), Logs when the change happened(timestamp), Identifies the type of action(INSERT,UPDATE, DELETE) 
ON OrderItems
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO OrderItems_AuditLog (
        OrderItemID,
        OldUnitPrice,
        NewUnitPrice,
        OldQuantity,
        NewQuantity,
        OldTotalAmount,
        NewTotalAmount,
        ActionType,
        ChangedBy,
        ChangedAt
    )
    SELECT
        COALESCE(i.OrderItemID, d.OrderItemID),
        d.UnitPrice,           -- Old price
        i.UnitPrice,           -- New price
        d.Quantity,            -- Old quantity
        i.Quantity,            -- New quantity
        d.TotalAmount,         -- Old total
        CASE 
            WHEN i.UnitPrice IS NOT NULL AND i.Quantity IS NOT NULL 
            THEN i.UnitPrice * i.Quantity 
            ELSE NULL 
        END,                   -- New total (calculated)
        CASE 
            WHEN i.OrderItemID IS NOT NULL AND d.OrderItemID IS NULL THEN 'INSERT'
            WHEN i.OrderItemID IS NOT NULL AND d.OrderItemID IS NOT NULL THEN 'UPDATE'
            WHEN i.OrderItemID IS NULL AND d.OrderItemID IS NOT NULL THEN 'DELETE'
        END,                   -- Action type
        SUSER_SNAME(),         -- Changed by (username)
        GETDATE()              -- Changed at (timestamp)
    FROM inserted i
    FULL OUTER JOIN deleted d ON i.OrderItemID = d.OrderItemID;
END;
GO


-- : UPDATE
UPDATE OrderItems
SET UnitPrice = 900.0, Quantity = 8
WHERE OrderItemID = 2;

--  INSERT
INSERT INTO OrderItems (ProductID, UnitPrice, Quantity)
VALUES (101, 15.00, 3);

-- Check the audit log
SELECT 
    AuditID,
    OrderItemID,
    OldUnitPrice,
    NewUnitPrice,
    OldQuantity,
    NewQuantity,
    ActionType,
    ChangedBy,
    ChangedAt
FROM OrderItems_AuditLog 
ORDER BY ChangedAt DESC;


SELECT * 
FROM OrderItems;

UPDATE OrderItems
SET UnitPrice = 198.00, Quantity = 8
WHERE OrderID = 2;

SELECT * 
FROM OrderItems_AuditLog  ORDER BY ChangedAt DESC;

SELECT 
    OrderItemID,
    OrderID,
    ProductID,
    UnitPrice,
    Quantity,
    TotalAmount
FROM OrderItems
ORDER BY OrderItemID DESC;


----Created an Audit table for Products tracker----
CREATE TABLE Products_AuditLog (
    AuditID INT IDENTITY(1,1) PRIMARY KEY,
    ProductID INT,
    
    -- Track Product Name changes
    OldProductName NVARCHAR(100) NULL,
    NewProductName NVARCHAR(100) NULL,
    
    -- Track Category changes
    OldCategory NVARCHAR(50) NULL,
    NewCategory NVARCHAR(50) NULL,
    
    -- Track Price changes
    OldPrice DECIMAL(18,2) NULL,
    NewPrice DECIMAL(18,2) NULL,
    
    -- Track Stock Quantity changes
    OldStockQty INT NULL,
    NewStockQty INT NULL,
    
    -- Audit metadata
    ActionType VARCHAR(20),              -- INSERT, UPDATE, DELETE
    ChangedBy VARCHAR(100) DEFAULT SUSER_SNAME(),
    ChangedAt DATETIME DEFAULT GETDATE()
);
GO

---Create Trigger to execute actions like tracking every changes automically logged who made the change ---
CREATE TRIGGER trg_Products_Audit
ON Products
AFTER INSERT, UPDATE, DELETE
AS
BEGIN
    SET NOCOUNT ON;

    INSERT INTO Products_AuditLog (
        ProductID,
        OldProductName,
        NewProductName,
        OldCategory,
        NewCategory,
        OldPrice,
        NewPrice,
        OldStockQty,
        NewStockQty,
        ActionType,
        ChangedBy,
        ChangedAt
    )
    SELECT
        COALESCE(i.ProductID, d.ProductID),
        d.ProductName,
        i.ProductName,
        d.Category,
        i.Category,
        d.Price,
        i.Price,
        d.StockQty,
        i.StockQty,
        CASE 
            WHEN i.ProductID IS NOT NULL AND d.ProductID IS NULL THEN 'INSERT'
            WHEN i.ProductID IS NOT NULL AND d.ProductID IS NOT NULL THEN 'UPDATE'
            WHEN i.ProductID IS NULL AND d.ProductID IS NOT NULL THEN 'DELETE'
        END,
        SUSER_SNAME(),
        GETDATE()
    FROM inserted i
    FULL OUTER JOIN deleted d ON i.ProductID = d.ProductID;
END;
GO

--  UPDATE iPhone 15 price
UPDATE Products
SET Price = 1300000.00, StockQty = 15
WHERE ProductID = 1;

--  UPDATE Dell Laptop category
UPDATE Products
SET Category = 'Electronics', Price = 900000.00
WHERE ProductID = 2;

--  INSERT a new product
INSERT INTO Products (ProductName, Category, Price, StockQty)
VALUES ('Samsung Galaxy', 'Electronics', 1100000.00, 20);

--  Check the audit log
SELECT * FROM Products_AuditLog ORDER BY ChangedAt DESC;
SELECT * 
FROM OrderItems_AuditLog  ORDER BY ChangedAt DESC;

SELECT *
FROM OrderItems
ORDER BY OrderItemID DESC;
