/* =====================================================
   T3 - Database (SQL Server) | Student Attendance
   Author : Kerolos Eissa
   Assumes existing tables: Students(StudentID), Courses(CourseID)
   (rename them below if the ERD uses different names)
   ===================================================== */

-- 1) Table -------------------------------------------------
IF OBJECT_ID('dbo.Attendance', 'U') IS NOT NULL DROP TABLE dbo.Attendance;
GO

CREATE TABLE dbo.Attendance (
    AttendanceID INT IDENTITY(1,1) NOT NULL,
    StudentID    INT          NOT NULL,
    CourseID     INT          NOT NULL,
    SessionDate  DATE         NOT NULL,
    Status       VARCHAR(10)  NOT NULL,
    Notes        NVARCHAR(200) NULL,
    CreatedAt    DATETIME2    NOT NULL CONSTRAINT DF_Attendance_CreatedAt DEFAULT SYSDATETIME(),

    CONSTRAINT PK_Attendance PRIMARY KEY (AttendanceID),
    CONSTRAINT FK_Attendance_Student FOREIGN KEY (StudentID) REFERENCES dbo.Students(StudentID),
    CONSTRAINT FK_Attendance_Course  FOREIGN KEY (CourseID)  REFERENCES dbo.Courses(CourseID),
    CONSTRAINT UQ_Attendance_Student_Course_Date UNIQUE (StudentID, CourseID, SessionDate),
    CONSTRAINT CK_Attendance_Status CHECK (Status IN ('Present', 'Absent', 'Late'))
);
GO

CREATE INDEX IX_Attendance_Course_Date ON dbo.Attendance (CourseID, SessionDate);
GO

-- 2) Stored procedures ------------------------------------
-- Mark (or update) attendance for one student in one session
CREATE OR ALTER PROCEDURE dbo.sp_MarkAttendance
    @StudentID   INT,
    @CourseID    INT,
    @SessionDate DATE,
    @Status      VARCHAR(10),
    @Notes       NVARCHAR(200) = NULL
AS
BEGIN
    SET NOCOUNT ON;

    IF @Status NOT IN ('Present', 'Absent', 'Late')
    BEGIN
        RAISERROR('Status must be Present, Absent or Late.', 16, 1);
        RETURN;
    END

    IF EXISTS (SELECT 1 FROM dbo.Attendance
               WHERE StudentID = @StudentID AND CourseID = @CourseID AND SessionDate = @SessionDate)
        UPDATE dbo.Attendance
        SET Status = @Status, Notes = @Notes
        WHERE StudentID = @StudentID AND CourseID = @CourseID AND SessionDate = @SessionDate;
    ELSE
        INSERT INTO dbo.Attendance (StudentID, CourseID, SessionDate, Status, Notes)
        VALUES (@StudentID, @CourseID, @SessionDate, @Status, @Notes);
END
GO

-- All attendance records of one student (optionally one course)
CREATE OR ALTER PROCEDURE dbo.sp_GetAttendanceByStudent
    @StudentID INT,
    @CourseID  INT = NULL
AS
BEGIN
    SET NOCOUNT ON;
    SELECT AttendanceID, StudentID, CourseID, SessionDate, Status, Notes
    FROM dbo.Attendance
    WHERE StudentID = @StudentID
      AND (@CourseID IS NULL OR CourseID = @CourseID)
    ORDER BY SessionDate DESC;
END
GO

-- Attendance of a whole course (optionally one date)
CREATE OR ALTER PROCEDURE dbo.sp_GetAttendanceByCourse
    @CourseID    INT,
    @SessionDate DATE = NULL
AS
BEGIN
    SET NOCOUNT ON;
    SELECT AttendanceID, StudentID, CourseID, SessionDate, Status, Notes
    FROM dbo.Attendance
    WHERE CourseID = @CourseID
      AND (@SessionDate IS NULL OR SessionDate = @SessionDate)
    ORDER BY SessionDate DESC, StudentID;
END
GO

-- Attendance percentage per student in a course (useful for T8 Reports)
CREATE OR ALTER PROCEDURE dbo.sp_GetAttendanceSummary
    @CourseID INT
AS
BEGIN
    SET NOCOUNT ON;
    SELECT StudentID,
           COUNT(*)                                           AS TotalSessions,
           SUM(CASE WHEN Status IN ('Present','Late') THEN 1 ELSE 0 END) AS Attended,
           CAST(100.0 * SUM(CASE WHEN Status IN ('Present','Late') THEN 1 ELSE 0 END)
                / COUNT(*) AS DECIMAL(5,2))                   AS AttendancePercent
    FROM dbo.Attendance
    WHERE CourseID = @CourseID
    GROUP BY StudentID;
END
GO

-- 3) Seed data (needs StudentID 1,2 and CourseID 1 to exist) -
EXEC dbo.sp_MarkAttendance 1, 1, '2026-11-01', 'Present';
EXEC dbo.sp_MarkAttendance 2, 1, '2026-11-01', 'Absent', N'Sick leave';
EXEC dbo.sp_MarkAttendance 1, 1, '2026-11-08', 'Late';
EXEC dbo.sp_MarkAttendance 2, 1, '2026-11-08', 'Present';
GO

-- 4) Quick tests ------------------------------------------
EXEC dbo.sp_GetAttendanceByStudent @StudentID = 1;
EXEC dbo.sp_GetAttendanceByCourse  @CourseID = 1, @SessionDate = '2026-11-01';
EXEC dbo.sp_GetAttendanceSummary   @CourseID = 1;
GO
