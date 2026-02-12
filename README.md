# Call-Center Appointments API

## Overview

This API allows the Halo Hermina call-center application to create patient appointments in the Hermina hospital system. Appointments are forwarded to the Afya hospital information system (SIMRS) and persisted in the Hermina database upon success.

The request payload uses the same parameter format as the Afya API (PascalCase/camelCase keys).

# URL

- **Production:** <https://api.herminahospitals.com>
- **Staging:** <https://api.herminahospital.com>

## Authentication

All requests must include HTTP Basic Auth credentials via the `Authorization` header.

```
Authorization: Basic <base64(username:password)>
```

The call-center team needs an `ApiClient` record created with their `username` and `password`. Please contact Yana to obtain your credentials.

**Missing or invalid credentials will return `401 Unauthorized`.** Locked accounts will return `401` with an "account locked" message.

---

## Create Appointment

### `POST /api/v1/client/call-center/appointments`

### Headers

| Header          | Required | Description                                           |
| --------------- | -------- | ----------------------------------------------------- |
| `Authorization` | Yes      | HTTP Basic Auth (`Basic <base64(username:password)>`) |
| `Content-Type`  | Yes      | Must be `application/json`                            |

### Request Body

Parameters use the same naming convention as the Afya API.

| Field                  | Type    | Required | Description                                                                                   |
| ---------------------- | ------- | -------- | --------------------------------------------------------------------------------------------- |
| `DateAppointment`      | string  | Yes      | Appointment date (`YYYY-MM-DD`)                                                               |
| `ClinicId`             | integer | Yes      | Afya clinic/unit ID                                                                           |
| `DoctorId`             | integer | Yes      | Afya doctor code (maps to `hospital_doctors.doctor_code`, e.g., `51701`)                      |
| `FromTime`             | string  | Yes      | Appointment start time (ISO 8601 datetime, e.g., `"2025-10-15T15:15:00.000+07:00"`)           |
| `ToTime`               | string  | Yes      | Appointment end time (ISO 8601 datetime, e.g., `"2025-10-15T15:30:00.000+07:00"`)             |
| `DoctorScheduleDetKey` | string  | Yes      | Afya schedule detail key                                                                      |
| `IdentificationNo`     | string  | No       | Patient medical record number. Use empty string `""` or omit for new patients                 |
| `name`                 | string  | No\*     | Patient full name (\*required for new patients)                                               |
| `dob`                  | string  | No\*     | Patient date of birth (ISO 8601, e.g., `"1990-05-15"`) (\*required for new patients)          |
| `mobilePhone`          | string  | No\*     | Patient mobile phone number (e.g., `"081234567890"`) (\*required for new patients)            |
| `Gender`               | string  | No       | Patient gender: `"L"` (male) or `"P"` (female)                                                |
| `Notes`                | string  | No       | Appointment notes. Defaults to `"Appointment via Halo Hermina (call-center)"` if not provided |
| `guarantorId`          | string  | No       | Guarantor ID for the appointment                                                              |

**Important**: `DoctorId` and `ClinicId` are Afya system identifiers, not Hermina internal IDs. These are the same IDs returned by the Afya schedule/slot endpoints.

**Note**: `hospital_id` is NOT required. The hospital is automatically resolved from the `DoctorId` via the `hospital_doctors.doctor_code` mapping.

### Patient Lookup Logic

The system automatically finds or creates a patient account:

1. **If `IdentificationNo` is provided** (non-empty): Searches for an existing patient by medical record number.
2. **If `IdentificationNo` is empty or omitted**: Searches by `dob` + `mobilePhone` combination.
3. **If no match found**: A new patient account is created automatically with `provider: call_center`.

### Example Request (Existing Patient)

```bash
curl -X POST https://api.herminahospitals.com/api/v1/client/call-center/appointments \
  --user "your-username:your-password" \
  -H "Content-Type: application/json" \
  -d '{
    "IdentificationNo": "1490002838",
    "Notes": "",
    "guarantorId": "",
    "DateAppointment": "2025-10-15",
    "ClinicId": 3,
    "DoctorId": 51701,
    "FromTime": "2025-10-15T15:15:00.000+07:00",
    "ToTime": "2025-10-15T15:30:00.000+07:00",
    "DoctorScheduleDetKey": "10"
  }'
```

### Example Request (New Patient)

```bash
curl -X POST https://api.herminahospitals.com/api/v1/client/call-center/appointments \
  --user "your-username:your-password" \
  -H "Content-Type: application/json" \
  -d '{
    "IdentificationNo": "",
    "Notes": "",
    "guarantorId": "",
    "DateAppointment": "2025-10-15",
    "ClinicId": 3,
    "DoctorId": 51701,
    "FromTime": "2025-10-15T18:45:00.000+07:00",
    "ToTime": "2025-10-15T18:50:00.000+07:00",
    "DoctorScheduleDetKey": "8",
    "name": "Jane Smith",
    "dob": "1985-03-20",
    "mobilePhone": "081987654321"
  }'
```

---

### Success Response

**Status: `200 OK`**

```json
{
  "success": true,
  "code": 200,
  "status": "OK",
  "message": "Appointment created successfully",
  "alert": "success",
  "data": {
    "id": "a1b2c3d4-uuid",
    "type": "appointment",
    "attributes": {
      "id": 12345,
      "date": "2025-10-15T00:00:00.000+07:00",
      "time": "15:15-15:30",
      "status": "upcoming",
      "queue_number": 5,
      "mrn": "1490002838",
      "medical_record_number": "1490002838",
      "appointment_method": "afya",
      "source": "call_center",
      "notes": "Patient complaint: headache",
      "hospital_name": "RS Hermina Jatinegara",
      "doctor_name": "Dr. Example, Sp.PD",
      "doctor_speciality_name": "Penyakit Dalam"
    }
  }
}
```

### Error Responses

**Status: `401 Unauthorized`** - Invalid or missing credentials

```json
{
  "success": false,
  "code": 401,
  "status": "Unauthorized",
  "message": "Invalid or missing credentials"
}
```

**Status: `422 Unprocessable Entity`** - Validation or Afya error

```json
{
  "success": false,
  "code": 422,
  "status": "Unprocessable Entity",
  "message": "Missing required parameters: DateAppointment, DoctorId"
}
```

```json
{
  "success": false,
  "code": 422,
  "status": "Unprocessable Entity",
  "message": "Unable to create appointment in hospital system",
  "simrs_data": {
    "metadata": { "code": 400, "message": "Detailed Afya error message" }
  }
}
```

### Possible Error Messages

| Message                                           | Cause                                            |
| ------------------------------------------------- | ------------------------------------------------ |
| `Missing required parameters: <field_list>`       | One or more required fields are missing          |
| `Doctor not found for the specified doctor code`  | No `hospital_doctor` with matching `doctor_code` |
| `Hospital does not support Afya appointments`     | Hospital not configured for Afya                 |
| `Afya API is not configured for this hospital`    | Hospital's Afya API URL not set                  |
| `Unable to create appointment in hospital system` | Afya returned an error                           |
| Afya-specific error messages                      | Validation errors from the hospital system       |
