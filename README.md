# Enterprise Employee Attendance Management System

A production-grade, full-stack Employee Attendance Management System built with **React (Vite + Tailwind CSS)**, **FastAPI (Python)**, and **MySQL 8.0**, featuring real-time attendance tracking, overnight shift engine, automated audit trails, and **Python Machine Learning & Statistical Anomaly Detection** using **Pandas, NumPy, and Scikit-learn (Isolation Forest)**.

---

## 1. Project Overview

The **Employee Attendance Management System** provides organizations with a centralized, data-driven platform to manage their workforce, schedule complex shifts (including day, evening, and overnight shifts), enforce punctuality policies, track real-time attendance, generate reports (CSV/PDF), and automatically flag unusual attendance behaviors.

### System Roles
- **ADMINISTRATOR**: Complete system oversight — manage employees, departments, shifts, and shift assignments; view real-time organization attendance; execute audited manual attendance adjustments; view statistical dashboards; run and resolve machine-learning anomaly alerts; export CSV/PDF reports.
- **EMPLOYEE**: Secure self-service portal — live digital check-in and check-out with automatic late and early-departure calculations; view assigned shift and grace periods; track personal attendance history, hours worked, and monthly punctuality rate.

---

## 2. Key Features

- **Relational Integrity & Normalized Database**: 8 normalized tables with foreign keys, cascade rules, unique indexes, and deletion constraints (preventing deletion of departments with active staff).
- **Overnight & Flexible Shift Engine**: Supports multi-day shifts (e.g. 21:00 to 05:00 next day) with configurable grace periods.
- **Biometric-Grade Check-In & Check-Out**:
  - Employee identity derived directly from JWT claims (prevents client spoofing).
  - Enforces single check-in session per day.
  - Prevents check-out without check-in or check-out preceding check-in.
  - Automatic calculation of late arrival minutes, early checkout minutes, and working hours.
- **Audited Manual Corrections**: Administrators can adjust attendance status or timestamps with mandatory audit justifications that record the modifier's user ID.
- **Reporting & Document Generation**:
  - Daily, weekly, monthly, employee-wise, and department-wise reports.
  - Streaming CSV download.
  - High-resolution PDF report generation via ReportLab.
- **Python Innovation: Anomaly Detection Engine**:
  - **Chronic Late Arrivals**: Detects recurring delays exceeding policy threshold in rolling 14-day windows.
  - **Frequent Early Checkouts**: Identifies persistent premature departures before shift completion.
  - **Consecutive Absenteeism**: Flags runs of 3+ consecutive unexcused absent days.
  - **Z-Score Working Hours Outliers**: Statistical detection of shift durations deviating significantly from the employee's historical baseline ($|Z| \ge 2.0$).
  - **Check-In Volatility**: Flags high standard deviation ($\sigma \ge 45\text{ min}$) indicating schedule instability.
  - **Multi-Variate Isolation Forest**: Detects non-linear multi-feature outliers across hours, delay, and departure.
  - Explainable alert descriptions with human review and resolution workflows.

---

## 3. Technology Stack

### Frontend
- **Framework**: React 19 + Vite
- **Styling**: Tailwind CSS
- **Routing**: React Router v7
- **HTTP Client**: Axios (configured with token injection and 401 interceptors)
- **Visualizations**: Recharts (Area charts, Bar charts, Pie charts)
- **Icons**: Lucide React

### Backend
- **Framework**: FastAPI (Python 3.10+)
- **ORM**: SQLAlchemy 2.0
- **Database Driver**: PyMySQL / Cryptography
- **Validation**: Pydantic v2 + Pydantic-Settings + Email-Validator
- **Authentication**: JWT (`python-jose`) with bcrypt password hashing
- **Reporting**: ReportLab (vector PDF generation) + Python CSV
- **Server**: Uvicorn ASGI

### Python Innovation / Data Science
- **Pandas**: Vectorized time-series aggregation and rolling attendance windows
- **NumPy**: Statistical variance, standard deviation, and Z-score calculations
- **Scikit-learn**: Isolation Forest multi-variate outlier detection

### Database
- **Engine**: MySQL 8.0 Community Server (InnoDB, utf8mb4)

---

## 4. Database Schema & Architecture

The database is normalized (`attendance_db`) with strict constraints and indexes:

1. **`departments`**: `department_id` (PK), `department_name` (UK), `description`, `created_at`.
2. **`employees`**: `employee_id` (PK), `employee_code` (UK), `first_name`, `last_name`, `email` (UK), `phone`, `department_id` (FK $\to$ `departments`, RESTRICT), `designation`, `date_of_joining`, `employment_status`, `created_at`.
3. **`users`**: `user_id` (PK), `username` (UK), `email` (UK), `password_hash`, `role` (`ADMIN`, `EMPLOYEE`), `employee_id` (FK $\to$ `employees`, SET NULL), `is_active`, `created_at`.
4. **`shifts`**: `shift_id` (PK), `shift_name` (UK), `start_time`, `end_time`, `grace_period_minutes`, `description`, `is_active`, `is_overnight`, `created_at`.
5. **`employee_shifts`**: `assignment_id` (PK), `employee_id` (FK), `shift_id` (FK), `effective_from`, `effective_to`, `created_at`.
6. **`attendance`**: `attendance_id` (PK), `employee_id` (FK), `attendance_date`, `check_in_time`, `check_out_time`, `status` (`Present`, `Late`, `Absent`, `Half Day`, `On Leave`), `total_working_hours`, `late_minutes`, `early_checkout_minutes`, `audit_remark`, `corrected_by` (FK $\to$ `users`), `created_at`. Compound UK: `(employee_id, attendance_date)`.
7. **`leaves`**: `leave_id` (PK), `employee_id` (FK), `start_date`, `end_date`, `leave_type`, `reason`, `approval_status`, `approved_by` (FK $\to$ `users`), `created_at`.
8. **`attendance_alerts`**: `alert_id` (PK), `employee_id` (FK), `attendance_id` (FK), `alert_type`, `description`, `severity` (`Low`, `Medium`, `High`, `Critical`), `detected_at`, `is_resolved`, `resolution_notes`, `resolved_at`.

---

## 5. ER Diagram

```mermaid
erDiagram
    DEPARTMENTS ||--o{ EMPLOYEES : "has"
    USERS |o--o| EMPLOYEES : "associated with"
    EMPLOYEES ||--o{ EMPLOYEE_SHIFTS : "assigned"
    SHIFTS ||--o{ EMPLOYEE_SHIFTS : "defines"
    EMPLOYEES ||--o{ ATTENDANCE : "logs"
    USERS |o--o{ ATTENDANCE : "audits/corrects"
    EMPLOYEES ||--o{ LEAVES : "requests"
    USERS |o--o{ LEAVES : "approves"
    EMPLOYEES ||--o{ ATTENDANCE_ALERTS : "triggers"
    ATTENDANCE |o--o{ ATTENDANCE_ALERTS : "associated with"

    DEPARTMENTS {
        int department_id PK
        varchar department_name UK
        text description
        timestamp created_at
    }

    EMPLOYEES {
        int employee_id PK
        varchar employee_code UK
        varchar first_name
        varchar last_name
        varchar email UK
        varchar phone
        int department_id FK
        varchar designation
        date date_of_joining
        enum employment_status
        timestamp created_at
    }

    USERS {
        int user_id PK
        varchar username UK
        varchar email UK
        varchar password_hash
        enum role
        int employee_id FK,UK
        boolean is_active
        timestamp created_at
    }

    SHIFTS {
        int shift_id PK
        varchar shift_name UK
        time start_time
        time end_time
        int grace_period_minutes
        varchar description
        boolean is_active
        boolean is_overnight
        timestamp created_at
    }

    EMPLOYEE_SHIFTS {
        int assignment_id PK
        int employee_id FK
        int shift_id FK
        date effective_from
        date effective_to
        timestamp created_at
    }

    ATTENDANCE {
        int attendance_id PK
        int employee_id FK
        date attendance_date
        datetime check_in_time
        datetime check_out_time
        enum status
        decimal total_working_hours
        int late_minutes
        int early_checkout_minutes
        varchar audit_remark
        int corrected_by FK
        timestamp created_at
    }

    LEAVES {
        int leave_id PK
        int employee_id FK
        date start_date
        date end_date
        enum leave_type
        text reason
        enum approval_status
        int approved_by FK
        timestamp created_at
    }

    ATTENDANCE_ALERTS {
        int alert_id PK
        int employee_id FK
        int attendance_id FK
        varchar alert_type
        text description
        enum severity
        timestamp detected_at
        boolean is_resolved
        text resolution_notes
        datetime resolved_at
    }
```

---

## 6. Sample Login Credentials

| Role | Username / Email | Password | Persona / Purpose |
| :--- | :--- | :--- | :--- |
| **Admin** | `admin@company.com` | `Admin@123` | Full administrative control, reports & anomalies |
| **Employee** | `john.doe@company.com` | `User@123` | Punctual employee (Engineering) |
| **Employee** | `jane.smith@company.com` | `User@123` | Active daytime worker |
| **Employee** | `michael.j@company.com` | `User@123` | Test Check-in & Check-out candidate |
| **Employee** | `robert.b@company.com` | `User@123` | Chronic Late Arrival profile |
| **Employee** | `emily.d@company.com` | `User@123` | Frequent Early Checkout profile |
| **Employee** | `lisa.a@company.com` | `User@123` | Night Shift with Volatile Working Hours |
| **Employee** | `james.t@company.com` | `User@123` | Consecutive Absenteeism profile |

*(Note: The login page includes 1-click quick fill buttons for instant testing).*

---

## 7. How to Run the Application on Windows

### Prerequisites
- Python 3.10+
- Node.js 18+ and npm
- MySQL Server 8.0 running locally on port 3306

---

### Step 1: Database Setup
1. Open PowerShell and import the database schema:
```powershell
Get-Content -Path 'd:\DBMS hackathon\database\schema.sql' -Raw | mysql -u root -pBangtan@11017
```
*(Replace `Bangtan@11017` with your local MySQL root password if different).*

2. Seed sample data and initial machine-learning anomaly alerts:
```powershell
cd 'd:\DBMS hackathon\backend'
python app/database/seed.py
```

---

### Step 2: Running the FastAPI Backend
```powershell
cd 'd:\DBMS hackathon\backend'
python -m uvicorn app.main:app --host 127.0.0.1 --port 8000 --reload
```
The backend API and Swagger UI will be accessible at:
- **API URL**: `http://127.0.0.1:8000`
- **Swagger Documentation**: `http://127.0.0.1:8000/docs`
- **ReDoc**: `http://127.0.0.1:8000/redoc`

---

### Step 3: Running the React Frontend
Open a new terminal window:
```powershell
cd 'd:\DBMS hackathon\frontend'
npm run dev -- --host 127.0.0.1 --port 5173
```
Open your browser and navigate to:
- **Application Portal**: `http://127.0.0.1:5173`

---

## 8. Running the Automated Test Suite

A comprehensive end-to-end integration test suite is included in `backend/tests/test_full_suite.py`:

```powershell
cd 'd:\DBMS hackathon\backend'
python tests/test_full_suite.py
```

The automated test suite verifies:
1. Health check endpoint (`200 OK`)
2. Bad password rejection (`401 Unauthorized`)
3. Admin JWT authentication & profile retrieval
4. Analytics dashboard KPIs and 7-day trend calculations
5. Relational constraint protection (cannot delete populated department)
6. Department CRUD lifecycle
7. Shift management with overnight spanning support
8. Employee creation with automated user credential provisioning
9. Employee role authentication
10. Strict role-based authorization boundary (`403 Forbidden` for non-admins)
11. Attendance Check-in lifecycle with duplicate check-in prevention (`400 Bad Request`)
12. Attendance Check-out lifecycle with working hour calculation and duplicate check-out prevention
13. Admin manual attendance correction with mandatory audit trail
14. Statistical & Machine Learning anomaly detection pipeline
15. Anomaly alert resolution workflow
16. Daily, weekly, and monthly attendance reporting
17. Real-time CSV export stream
18. Vectorized PDF export generation

---

## 9. Anomaly Detection Engine Details

The detection engine (`backend/app/anomaly_detection/detector.py`) executes 6 analytical algorithms over historical data:

1. **Chronic Late Arrivals**:
   - Analyzes the rolling 14-day attendance records.
   - Calculates total arrivals where `late_minutes > 10`.
   - Flags when delay count exceeds threshold ($\ge 3$ late sessions) with severity proportional to frequency.

2. **Frequent Early Checkouts**:
   - Detects workers departing prior to their shift completion without authorized half-day leave.
   - Flags occurrences where `early_checkout_minutes > 15` at least 3 times in 14 days.

3. **Consecutive Absenteeism**:
   - Identifies unbroken streaks of unexcused absent days ($\ge 3$ consecutive days).

4. **Working Hours Z-Score Outliers**:
   $$\mu = \frac{1}{N}\sum x_i, \quad \sigma = \sqrt{\frac{1}{N}\sum (x_i - \mu)^2}, \quad Z = \frac{x_i - \mu}{\sigma}$$
   - Identifies shift durations where $|Z| \ge 2.0$ (severely short or unusually long sessions).

5. **Schedule Volatility Index**:
   - Evaluates standard deviation of arrival minutes. Flags volatility if $\sigma \ge 45\text{ minutes}$, reflecting schedule instability.

6. **Isolation Forest Multi-Variate Detection**:
   - When $\ge 10$ historical sessions exist, fits an unsupervised Isolation Forest model over multi-dimensional feature vectors (working hours, arrival minute, delay, early departure) to flag complex non-linear outliers.

---

## 10. API Specification Summary

### Authentication (`/api/auth`)
- `POST /api/auth/login`: Authenticate username/email and password, returns JWT token and role.
- `POST /api/auth/register`: Create administrative or staff login.
- `GET /api/auth/me`: Retrieve current authenticated user profile.

### Employees (`/api/employees`)
- `GET /api/employees`: Query employees with search, department, and status filters.
- `POST /api/employees`: Create new employee with optional user account and initial shift.
- `GET /api/employees/{id}`: Detailed employee profile with attendance rates.
- `PUT /api/employees/{id}`: Update employee record.
- `DELETE /api/employees/{id}`: Soft deactivate employee.

### Departments (`/api/departments`)
- `GET /api/departments`: List all departments with active employee counts.
- `POST /api/departments`: Create department.
- `PUT /api/departments/{id}`: Update department.
- `DELETE /api/departments/{id}`: Delete department (protected against populated units).

### Shifts (`/api/shifts`)
- `GET /api/shifts`: List all shifts with overnight badges and active counts.
- `POST /api/shifts`: Create standard or overnight shift with grace period.
- `PUT /api/shifts/{id}`: Update shift.
- `DELETE /api/shifts/{id}`: Deactivate shift.
- `POST /api/shifts/assign`: Schedule employee shift with effective date range.
- `GET /api/shifts/assignments`: Query shift assignment history.

### Attendance (`/api/attendance`)
- `POST /api/attendance/check-in`: Employee check-in.
- `POST /api/attendance/check-out`: Employee check-out.
- `GET /api/attendance/today`: Live session status for employee dashboard.
- `GET /api/attendance/history`: Personal attendance history.
- `GET /api/attendance/all`: Admin master attendance log with full filters.
- `GET /api/attendance/{id}`: Single attendance record.
- `PUT /api/attendance/{id}`: Admin manual correction with mandatory audit remark.

### Reports (`/api/reports`)
- `GET /api/reports/daily`: Daily summary and records.
- `GET /api/reports/weekly`: 7-day summary.
- `GET /api/reports/monthly`: Monthly summary.
- `GET /api/reports/employee`: Employee-specific attendance range report.
- `GET /api/reports/department`: Department-specific attendance range report.
- `GET /api/reports/export/csv`: Download report as CSV.
- `GET /api/reports/export/pdf`: Download report as formatted PDF.

### Analytics (`/api/analytics`)
- `GET /api/analytics/dashboard`: KPI metrics, 7-day trend chart data, department breakdown, status pie chart.
- `GET /api/analytics/attendance-trends`: Historical trend data.
- `GET /api/analytics/department-attendance`: Departmental compliance data.

### Anomaly Detection (`/api/anomalies`)
- `GET /api/anomalies`: List alerts with severity and resolution filters.
- `GET /api/anomalies/{id}`: Detailed anomaly alert information.
- `PUT /api/anomalies/{id}/resolve`: Resolve anomaly alert with management notes.
- `POST /api/anomalies/detect`: Execute machine learning detection engine on demand.

---

## 11. Environment Variables Configuration

### Backend (`backend/.env`)
```ini
DATABASE_URL=mysql+pymysql://root:Bangtan%4011017@localhost:3306/attendance_db
SECRET_KEY=e83c748c9df10a56bd12b84f3c7e99da54a8b7c210d3e5f6a9810234bc567def
ALGORITHM=HS256
ACCESS_TOKEN_EXPIRE_MINUTES=1440
ORGANIZATION_TIMEZONE=Asia/Kolkata
GRACE_PERIOD_MINUTES_DEFAULT=15
ANOMALY_ZSCORE_THRESHOLD=2.0
MIN_RECORDS_FOR_ML=5
CHRONIC_LATE_COUNT_THRESHOLD=3
CHRONIC_EARLY_DEPARTURE_COUNT=3
CORS_ORIGINS=["http://localhost:5173", "http://127.0.0.1:5173", "http://localhost:3000"]
```

### Frontend (`frontend/.env`)
```ini
VITE_API_BASE_URL=http://localhost:8000/api
```
