# 5. Backend Schema Document

## Document Info
| Field | Value |
|-------|-------|
| Project name | _____ |
| Document owner | Gayatri |
| Version | 0.1 |
| Date | _____ |
| Status | Draft / In review / Approved |
| Database (from TRD) | _____ |

## 1. Overview
- Short description of the data model: _____

## 2. Entities / Tables
### TBL-01: _____ (e.g. users)
| Field | Type | Required | Key (PK/FK/Unique) | Default | Description |
|-------|------|----------|--------------------|---------|-------------|
| id | _____ | Yes | PK | _____ | _____ |
| _____ | _____ | _____ | _____ | _____ | _____ |
| created_at | timestamp | Yes | | now() | _____ |

### TBL-02: _____
| Field | Type | Required | Key | Default | Description |
|-------|------|----------|-----|---------|-------------|
| _____ | _____ | _____ | _____ | _____ | _____ |

## 3. Relationships
| From table | To table | Type (1-1 / 1-N / N-N) | Via field |
|------------|----------|------------------------|-----------|
| _____ | _____ | _____ | _____ |

## 4. ER Diagram
```
[TBL-01 ____] 1 ---- N [TBL-02 ____]
```
(or link: _____)

## 5. API Endpoints
| Method | Endpoint | Description | Auth | Request body | Response |
|--------|----------|-------------|------|--------------|----------|
| GET | /api/_____ | _____ | _____ | _____ | _____ |
| POST | /api/_____ | _____ | _____ | _____ | _____ |

## 6. Access Rules / Permissions
| Table / Endpoint | Who can read | Who can write |
|------------------|--------------|---------------|
| _____ | _____ | _____ |

## 7. Validation & Business Rules
- _____

## 8. Seed / Test Data
- _____

## 9. Open Questions
- [ ] _____

## Change Log
| Version | Date | Author | Change |
|---------|------|--------|--------|
| 0.1 | _____ | _____ | Template created |
