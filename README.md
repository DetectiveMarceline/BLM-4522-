# BLM-4522-Veritabanı Yedekleme ve Felaketten Kurtarma Planı

# ADIM 1: Veritabanı Durumunun Değerlendirilmesi
# Bu aşamada, SalesDB veritabanının mevcut durumu ve kurtarma modeli incelenmiştir. Veritabanının yedekleme geçmişi ve özellikleri kontrol edilmiştir.
# ADIM 2: Kurtarma Modelinin FULL Olarak Ayarlanması

ALTER DATABASE SalesDB SET RECOVERY FULL;
GO

# ADIM 3: Tam Yedek (Full Backup) Oluşturulması

BACKUP DATABASE SalesDB
TO DISK = 'C:\Temp\SalesDB_Full.bak'
WITH INIT, 
NAME = 'SalesDB-Full Backup',
DESCRIPTION = 'Full backup for SalesDB';
GO

# ADIM 4: Test İçin Veri Değişikliklerinin Yapılması

USE SalesDB;
GO

-- Yeni bir müşteri ekle
INSERT INTO dbo.Customers (
    FirstName, 
    MiddleInitial, 
    LastName
) 
VALUES (
    'Test', 
    'X', 
    'Customer'
);
GO

# ADIM 5: Fark Yedeği (Differential Backup) Oluşturulması

BACKUP DATABASE SalesDB
TO DISK = 'C:\Temp\SalesDB_Diff.bak'
WITH DIFFERENTIAL, 
NAME = 'SalesDB-Differential Backup',
DESCRIPTION = 'Differential backup after adding test customer';
GO

# ADIM 6: İşlem Günlüğü Yedeği (Transaction Log Backup) Alınması

BACKUP LOG SalesDB
TO DISK = 'C:\Temp\SalesDB_Log.trn'
WITH 
NAME = 'SalesDB-Transaction Log Backup',
DESCRIPTION = 'Transaction log backup after customer update';
GO

# ADIM 7: Kaza ile Silinen Verilerin Simülasyonu (Disaster Simulation)

USE SalesDB;
GO

-- Veri silme işlemi
BEGIN TRANSACTION;
DELETE FROM dbo.Customers WHERE FirstName = 'Test' AND LastName = 'Customer';
COMMIT;
GO

# ADIM 8: Zamana Bağlı Geri Yükleme (Point-in-Time Recovery)

-- Son log yedeği (tail-log backup)
BACKUP LOG SalesDB 
TO DISK = 'C:\Temp\SalesDB_TailLog.trn'
WITH NO_TRUNCATE;
GO

-- Tam yedeği geri yükleme
RESTORE DATABASE SalesDB_Recovered
FROM DISK = 'C:\Temp\SalesDB_Full.bak'
WITH 
MOVE 'SalesDBData' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\SalesDB_Recovered.mdf',
MOVE 'SalesDBLog' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\SalesDB_Recovered_log.ldf',
NORECOVERY, REPLACE;
GO

-- Fark yedeği uygulama
RESTORE DATABASE SalesDB_Recovered
FROM DISK = 'C:\Temp\SalesDB_Diff.bak'
WITH NORECOVERY;
GO

-- İşlem günlüğü yedeği uygulama
RESTORE LOG SalesDB_Recovered
FROM DISK = 'C:\Temp\SalesDB_Log.trn'
WITH RECOVERY;
GO

# ADIM 9: Otomatik Yedekleme için SQL Server Agent Job Oluşturulması

# ADIM 10: Yedeklerin Doğruluğunun Test Edilmesi

-- Yedeğin doğruluğunu kontrol etme
RESTORE VERIFYONLY
FROM DISK = 'C:\Temp\SalesDB_Full.bak';
GO

-- Test amaçlı farklı bir veritabanına geri yükleme
RESTORE DATABASE SalesDB_Test
FROM DISK = 'C:\Temp\SalesDB_Full.bak'
WITH 
MOVE 'SalesDBData' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\SalesDB_Test.mdf',
MOVE 'SalesDBLog' TO 'C:\Program Files\Microsoft SQL Server\MSSQL16.MSSQLSERVER\MSSQL\DATA\SalesDB_Test_log.ldf',
RECOVERY, REPLACE;
GO

