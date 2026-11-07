# Requirements Document: Employee Application

## Introduction

The employee interface is part of the same Cashier & Business Management Application. It allows
employees to manage their daily work activities, process sales, record attendance, and access their
employment information. The available features depend on the employee's assigned role and
permissions.

This document captures the functional and non-functional requirements for the employee side of the
application, expressed as user stories with EARS-formatted acceptance criteria.

## Requirements

### Requirement 1: Employee Dashboard

**User Story:** As an employee, I want a personalised dashboard, so that I can see my profile,
schedule, attendance, and notifications at a glance.

#### Acceptance Criteria

1. WHEN an employee opens the dashboard THEN the system SHALL display their personal profile and employment information.
2. WHEN an employee opens the dashboard THEN the system SHALL display their assigned outlet and job position.
3. WHEN an employee opens the dashboard THEN the system SHALL display the current work schedule and upcoming shifts.
4. WHEN an employee opens the dashboard THEN the system SHALL display the attendance status for the current day.
5. WHEN an employee opens the dashboard THEN the system SHALL display recent activities and important notifications.
6. IF the employee has pending tasks or approvals THEN the system SHALL display them on the dashboard.

### Requirement 2: Attendance & Timesheet

**User Story:** As an employee, I want to record and review my attendance, so that my working hours
are tracked accurately.

#### Acceptance Criteria

1. WHEN an employee clocks in or clocks out THEN the system SHALL record the timestamp against their account.
2. WHEN an employee views attendance THEN the system SHALL display daily attendance records.
3. WHEN an employee views attendance THEN the system SHALL display total working hours.
4. WHEN an employee views attendance THEN the system SHALL display late arrivals, absences, and overtime records.
5. WHEN an employee requests a timesheet THEN the system SHALL display weekly and monthly timesheets.
6. WHEN an employee submits an attendance correction request THEN the system SHALL record the request for approval.
7. WHEN an employee submits a leave or absence request THEN the system SHALL record the request for approval.
8. WHEN an employee views their requests THEN the system SHALL display the status of each submitted request.

### Requirement 3: Work Schedule

**User Story:** As an employee, I want to view and manage my work schedule, so that I know when I am
expected to work and can request changes.

#### Acceptance Criteria

1. WHEN an employee views the schedule THEN the system SHALL display assigned working schedules and shifts.
2. WHEN an employee views the schedule THEN the system SHALL display upcoming shifts and working days.
3. WHEN a schedule change affects an employee THEN the system SHALL notify the employee.
4. WHEN a shift swap or schedule adjustment is approved THEN the system SHALL display it in the employee's schedule.
5. WHERE shift change requests are permitted THEN the system SHALL allow the employee to submit a shift change request.

### Requirement 4: Point of Sale (Cashier)

**User Story:** As a cashier, I want to process sales and manage my cashier shift, so that I can serve
customers at my assigned outlet.

#### Acceptance Criteria

1. WHEN a cashier logs in with an assigned employee account THEN the system SHALL grant access to the POS.
2. WHEN a cashier opens the POS THEN the system SHALL display products available at the assigned outlet.
3. WHEN a cashier searches for a product THEN the system SHALL support search by name, SKU, or barcode.
4. WHEN a cashier creates a transaction THEN the system SHALL process the customer transaction.
5. WHERE the cashier is authorised THEN the system SHALL allow applying discounts.
6. WHEN a cashier takes payment THEN the system SHALL accept supported payment methods.
7. WHEN a transaction completes THEN the system SHALL generate a receipt that can be printed or shared.
8. WHEN a cashier views transaction history THEN the system SHALL display their personal transaction history.
9. WHERE the cashier is authorised THEN the system SHALL allow processing refunds or cancellations.
10. WHEN a cashier opens or closes a shift THEN the system SHALL record the shift event.
11. WHEN a cashier opens or closes a shift THEN the system SHALL record opening and closing cash balances.

### Requirement 5: Payroll & Salary Information

**User Story:** As an employee, I want to view my salary and payroll information, so that I understand
my earnings and payment history.

#### Acceptance Criteria

1. WHEN an employee views payroll THEN the system SHALL display current salary information.
2. WHEN an employee views payroll THEN the system SHALL display monthly payroll summaries.
3. WHEN an employee views a payroll summary THEN the system SHALL display the salary breakdown including basic salary, allowances, bonuses, overtime, and deductions.
4. WHEN an employee requests a payslip THEN the system SHALL allow viewing and downloading individual payslips.
5. WHEN an employee views payment history THEN the system SHALL display salary payment history.
6. WHEN an employee views payments THEN the system SHALL display pending and completed salary payments.

### Requirement 6: Leave Management

**User Story:** As an employee, I want to request and track leave, so that I can manage my time off.

#### Acceptance Criteria

1. WHEN an employee submits an annual leave request THEN the system SHALL record the request for approval.
2. WHEN an employee submits a sick leave or other absence request THEN the system SHALL record the request for approval.
3. WHEN an employee submits a leave request THEN the system SHALL allow specifying leave dates and attaching supporting documents where required.
4. WHEN an employee views leave information THEN the system SHALL display leave balances.
5. WHEN an employee views a leave request THEN the system SHALL display its approval or rejection status.
6. WHEN an employee views leave history THEN the system SHALL display historical leave records.

### Requirement 7: Notifications

**User Story:** As an employee, I want to receive relevant notifications, so that I stay informed about
my work and employment.

#### Acceptance Criteria

1. WHEN an attendance action is due THEN the system SHALL send an attendance reminder.
2. WHEN a work schedule is updated THEN the system SHALL notify the employee.
3. WHEN a leave request status changes THEN the system SHALL notify the employee.
4. WHEN a payroll or salary payment occurs THEN the system SHALL notify the employee.
5. WHEN management publishes an important announcement THEN the system SHALL notify the employee.

### Requirement 8: Employee Profile

**User Story:** As an employee, I want to view and maintain my profile, so that my information and
credentials stay accurate and secure.

#### Acceptance Criteria

1. WHEN an employee views their profile THEN the system SHALL display personal and employment information.
2. WHERE fields are editable THEN the system SHALL allow the employee to update permitted personal information.
3. WHEN an employee changes their password THEN the system SHALL update the account password.
4. WHEN an employee views their profile THEN the system SHALL display the assigned role and outlet.
5. WHERE employment history is available THEN the system SHALL display it.

### Requirement 9: Access & Permissions

**User Story:** As the business, I want employee access to be restricted by role, so that employees
only see and act on what they are permitted to.

#### Acceptance Criteria

1. THE system SHALL restrict each employee to their own personal, attendance, timesheet, leave, and payroll information.
2. WHERE a cashier has POS permissions THEN the system SHALL grant POS features according to those permissions.
3. THE system SHALL prevent employees from accessing business-wide sales, accounting, financial reports, or other employees' payroll information.
4. THE system SHALL prevent employees from directly modifying approved attendance records or payroll information.
5. WHERE an employee is a manager THEN the system SHALL grant additional permissions according to their assigned role.
6. WHEN an employee performs a sensitive action THEN the system SHALL record it in the system activity log.

### Requirement 10: Non-Functional Requirements

**User Story:** As an employee, I want the application to be reliable, fast, and secure, so that I can
depend on it during my workday.

#### Acceptance Criteria

1. THE system SHALL provide a simple and accessible interface suitable for employees with limited technical experience (Usability).
2. WHEN an employee performs attendance recording or POS operations THEN the system SHALL respond quickly (Performance).
3. THE system SHALL ensure employees only access information and functions permitted by their roles (Security).
4. THE system SHALL make schedules, attendance, and employment information available when needed (Availability).
5. THE system SHALL synchronise employee activities with the central business database (Data Synchronisation).
