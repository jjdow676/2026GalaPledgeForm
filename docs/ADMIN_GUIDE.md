# 2026 Bridges Gala Pledge Form - Administrator Guide

## Overview

The 2026 Bridges Gala Sponsor Pledge Form is a web application that allows sponsors to submit their sponsorship pledges for the annual Bridges Gala event. This guide covers system administration, configuration, and troubleshooting.

## Architecture

### Components

| Component | Location | Technology | Hosting |
|-----------|----------|------------|---------|
| Frontend (Pledge Form) | `2026GalaPledgeForm` | React 18, Tailwind CSS | Azure Static Web Apps |
| Backend API | `gala-pledge-backend` | Node.js, Express | Azure App Service |
| Payment Portal | `2026GalaFrontEnd` | Stripe Integration | Separate deployment |

### URLs

| Environment | URL |
|-------------|-----|
| Production Frontend | https://pledge.bridgestowork.org |
| Production API | https://gala-pledge-backend-h0arg5f8g8e2hwf6.eastus2-01.azurewebsites.net |
| Local Development | http://localhost:5500 (frontend), http://localhost:8080 (backend) |

---

## Email Configuration

### Internal Recipients

The following team members receive notification emails when a pledge is submitted:

| Name | Email | Role |
|------|-------|------|
| Ann Hearn | Ann.Hearn@bridgestowork.org | Bridges Staff |
| Kate Brown | Kate.Brown@bridgestowork.org | Bridges Staff |
| Justin Down | Justin.Down@bridgestowork.org | Bridges Staff |
| Gala Team | bridgesgala@orrgroup.com | ORR Group (Event Coordinator) |

To modify recipients, edit the `INTERNAL_RECIPIENTS` array in `gala-pledge-backend/server.js`.

### Email Types

| Email Type | Recipient | Trigger | Attachment |
|------------|-----------|---------|------------|
| Sponsor Confirmation | Sponsor's email address | Successful pledge submission | PDF pledge summary |
| Internal Notification | All internal recipients | Successful pledge submission | PDF pledge summary |

### SendGrid Configuration

The backend uses SendGrid for email delivery.

| Setting | Value |
|---------|-------|
| Environment Variable | `SENDGRID_API_KEY` |
| From Address | bridgesgala@orrgroup.com |
| From Name | 2026 Bridges Gala |

---

## Sponsorship Tiers

| Tier | Amount | Guests | VIP Reception |
|------|--------|--------|---------------|
| Champion | $75,000 | 30 | Yes |
| Premier | $50,000 | 20 | Yes |
| Leadership Circle | $35,000 | 10 | Yes |
| Partner | $17,500 | 10 | No (General Reception) |
| Benefactor | $12,500 | 10 | No (General Reception) |
| Individual Ticket | $1,500/ticket | 1 per ticket | No (General Reception) |
| Donation | Custom | N/A | N/A |

To modify tiers, edit the `TIER_BENEFITS` object in `2026GalaPledgeForm/index.html`.

---

## Payment Options

| Option ID | Display Label | Instructions Sent |
|-----------|---------------|-------------------|
| `check` | Check (Payable to: Bridges from School to Work) | Mailing address provided |
| `credit_card` | Credit Card (via gala.bridgestowork.org) | Link to payment portal |
| `wire_stock` | Wire or Stock Transfer | Contact info for arrangements |
| `invoice` | Request an Invoice | Invoice will be sent |

---

## Data Flow

```
1. Sponsor fills out form on pledge.bridgestowork.org
2. Frontend validates all required fields
3. POST request sent to /submit-pledge endpoint
4. Backend validates data and generates PDF
5. SendGrid sends confirmation to sponsor
6. SendGrid sends notification to internal team
7. Sponsor redirected to thank-you.html
```

---

## Pledge Data Structure

The following data is collected and stored for each pledge:

| Field | Required | Description |
|-------|----------|-------------|
| `tierLabel` | Yes | Selected sponsorship tier |
| `amount` | Yes | Total pledge amount in USD |
| `quantity` | No | Number of tickets (Individual Ticket only) |
| `companyOrg` | Yes | Company or organization name |
| `sponsorshipName` | Yes | Name for event materials |
| `primaryContact` | Yes | Primary contact person |
| `title` | No | Contact's job title |
| `streetAddress` | Yes | Mailing address |
| `city` | Yes | City |
| `state` | Yes | US state abbreviation |
| `zip` | Yes | ZIP code |
| `email` | Yes | Primary email address |
| `phone` | Yes | Phone number |
| `paymentContactName` | No | Alternate payment contact |
| `paymentContactEmail` | No | Alternate payment email |
| `paymentOption` | Yes | Selected payment method |
| `pledgeDate` | Auto | Date of submission |

---

## Environment Variables

### Backend (`gala-pledge-backend`)

| Variable | Required | Description |
|----------|----------|-------------|
| `SENDGRID_API_KEY` | Yes | SendGrid API key for email delivery |
| `PORT` | No | Server port (default: 8080) |

### Frontend

The frontend uses no environment variables. API endpoint is configured in `index.html`:

```javascript
const API_BASE = isLocal
  ? "http://localhost:8080"
  : "https://gala-pledge-backend-h0arg5f8g8e2hwf6.eastus2-01.azurewebsites.net";
```

---

## Deployment

### Frontend Deployment

The frontend deploys automatically via GitHub Actions to Azure Static Web Apps when changes are pushed to the main branch.

**Workflow file:** `.github/workflows/azure-static-web-apps-*.yml`

### Backend Deployment

The backend deploys automatically via GitHub Actions to Azure App Service when changes are pushed to the master branch.

**Workflow file:** `.github/workflows/master_gala-pledge-backend.yml`

**Azure App Service:** `gala-pledge-backend` (East US 2)

---

## CORS Configuration

The backend allows requests from:

| Origin | Purpose |
|--------|---------|
| https://pledge.bridgestowork.org | Production frontend |
| http://localhost:5500 | Local development (VS Code Live Server) |
| http://localhost:3000 | Local development (React dev server) |
| http://127.0.0.1:5500 | Local development (alternate) |

To add origins, modify the CORS configuration in `server.js`.

---

## Troubleshooting

### Common Issues

| Issue | Possible Cause | Solution |
|-------|----------------|----------|
| Emails not sending | Invalid SendGrid API key | Verify `SENDGRID_API_KEY` in Azure App Service configuration |
| CORS errors | Origin not whitelisted | Add origin to CORS configuration in `server.js` |
| Form submission fails | Backend not responding | Check Azure App Service health and logs |
| PDF not generating | pdfkit error | Check server logs for PDF generation errors |

### Checking Logs

**Azure App Service Logs:**
1. Go to Azure Portal > App Services > gala-pledge-backend
2. Navigate to Monitoring > Log stream
3. Or use: Diagnose and solve problems > Application Logs

### Testing Locally

1. Clone both repositories
2. Set up backend:
   ```bash
   cd gala-pledge-backend
   cp .env.example .env
   # Add your SendGrid API key to .env
   npm install
   npm start
   ```
3. Open frontend with Live Server on port 5500

---

## Security Considerations

- All user input is HTML-escaped before being included in emails or PDFs
- CORS restricts API access to approved origins
- No sensitive data (passwords, payment info) is collected or stored
- Payment processing is handled separately via Stripe

---

## Contact Information

| Role | Contact |
|------|---------|
| Gala Team Email | BridgesGala@orrgroup.com |
| Gala Team Phone | (202) 719-8053 |
| Technical Support | Justin.Down@bridgestowork.org |

---

## Event Details

| Detail | Value |
|--------|-------|
| Event Date | Monday, June 22, 2026 |
| Location | Marriott Marquis, Washington, DC |
| Schedule | 6PM Reception, 7PM Dinner & Program, 9PM Afterglow |
| Organization | Bridges from School to Work |
| Tax ID | 52-1655740 (501c3) |
